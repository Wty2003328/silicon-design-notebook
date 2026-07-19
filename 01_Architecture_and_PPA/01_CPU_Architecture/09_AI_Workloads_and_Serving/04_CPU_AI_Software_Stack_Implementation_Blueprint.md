# CPU Artificial-Intelligence Software-Stack Implementation Blueprint

> **Abbreviation key:** central processing unit (CPU); artificial intelligence (AI); intermediate representation (IR); application binary interface (ABI); just-in-time compilation (JIT); non-uniform memory access (NUMA); single instruction, multiple data (SIMD); input/output (I/O); graphics processing unit (GPU); neural processing unit (NPU).

## 0. Purpose and design ideology

This chapter reconstructs the software between a model artifact and CPU instructions or accelerator submissions. The CPU AI stack owns more than arithmetic: model validation, graph capture, tensor metadata, operator dispatch, weight transformation, memory placement, thread pools, kernel selection, dynamic-shape handling, fallback, device submission, and completion.

The design ideology is **keep semantics stable while specializing execution**. Framework-visible tensors and operators define meaning. Compilation may fuse, reorder, quantize, pack, parallelize, or offload only under explicit legality rules. Every optimized plan retains a route to a reference implementation and an identity that connects a graph node to generated instructions and measured work.

Read [AI Operators on CPU Microarchitecture](02_AI_Operators_on_CPU_Microarchitecture.md) for operator mechanisms and [CPU Implementation Blueprints](../10_Implementation_Blueprints/00_Index.md) for hardware contracts.

## 1. Layer and ownership map

~~~mermaid
flowchart LR
    ART["model + tokenizer/feature artifacts"] --> LOAD["loader + validator"]
    LOAD --> GRAPH["semantic graph + tensor IR"]
    GRAPH --> PLAN["partition + fusion + layout + precision plan"]
    PLAN --> EXEC["execution-plan registry"]
    EXEC --> DISP["operator/kernel dispatcher"]
    DISP --> CPU["CPU kernel + thread pool"]
    DISP --> DEV["GPU/NPU runtime submission"]
    MEM["allocator + NUMA placement + weight cache"] --> EXEC
    MEM --> CPU
    MEM --> DEV
~~~

Assign one owner to every object:

- artifact loader owns integrity, schema/version, tensor index, and immutable source bytes;
- graph importer owns framework-to-IR semantic translation;
- compiler owns transformed graph and proof/diagnostic for every rewrite;
- plan registry owns executable identity and compatibility;
- runtime owns live tensor buffers, tasks, dependencies, and completion;
- kernel library/JIT owns target-specific instructions and required packed layout;
- device adapter owns accelerator context, memory registration, submissions, and faults.

Do not let a kernel silently reinterpret tensor layout or quantization. Layout/scale become typed plan properties checked at dispatch.

## 2. Artifact and tensor schemas

A model manifest stores model/version, graph format, tensor shards, byte offsets/lengths, element types, logical shapes, physical layouts, quantization scales/zero points or group metadata, checksums, tokenizer/feature schema, required operators, minimum runtime version, and optional signature. Tensor data is immutable after verification; transformed/packed copies get separate identities.

A runtime tensor descriptor needs storage object and generation, byte range, element type, shape, strides/layout, device and NUMA node, alignment, read/write/alias attributes, quantization metadata, lifetime class, and readiness event. Views share storage but change shape/stride; ownership and alias analysis must prevent a supposedly out-of-place kernel overwriting a live input.

An execution-plan manifest records source artifact hash, compiler/build configuration, CPU instruction extensions, kernel-library version, graph partitions, shape profiles, packed-weight objects, workspace bound, thread/NUMA assumptions, device requirements, and validation signature. A plan compiled for unavailable instructions must be rejected or use a compatible variant.

## 3. Graph import and compiler pipeline

The CPU compiler pipeline is causal:

1. parse and validate graph/operator schemas and constants;
2. infer static or symbolic shapes, types, layouts, aliases, and side effects;
3. canonicalize many framework operations into a smaller semantic operator set;
4. partition unsupported or accelerator-preferred regions while preserving cross-region ordering;
5. choose precision/quantization only with a calibration/quality contract;
6. fuse operations when data dependencies, aliasing, numerical order, and dynamic-shape rules permit;
7. choose layouts and prepack immutable weights for candidate kernels;
8. tile and parallelize using cache, vector/matrix, core, and NUMA cost models;
9. allocate/reuse temporary buffers from proven liveness intervals;
10. emit execution tasks, dependencies, kernel variants, fallbacks, and diagnostics.

The IR must store operation identity, inputs/outputs, shapes/strides, type/precision, constants, side effects, alias sets, device/NUMA placement, source mapping, and dynamic guards. Each pass declares preconditions and postconditions. A dynamic guard chooses a specialized plan only when runtime shape/alignment/layout and CPU-feature conditions hold; failure selects another plan rather than executing illegally.

## 4. Kernel dispatch and ABI

A kernel ABI defines operation, tensor descriptors, scalar attributes, workspace pointer/size, thread-team context, cancellation/fault status, and completion semantics. For every variant, registry metadata records supported shapes/layouts/types, required ISA extensions, alignment, scratch/workspace, thread scaling range, deterministic/numerical behavior, and cost-model coefficients.

Dispatch is a constrained selection:

1. filter variants by semantic legality and hardware features;
2. filter by shape, stride, alignment, precision, workspace, and determinism;
3. estimate cost from calibrated regimes or retrieve an autotuned result;
4. reserve workspace/thread resources;
5. invoke the chosen kernel and record variant identity;
6. fall back on a safe reference implementation for unsupported cases.

Autotuning keys include operation dimensions, layout, precision, CPU model/features, core/NUMA allocation, thread count, and relevant frequency/power mode. Cache results with compiler/kernel versions. Benchmarking one candidate changes cache and frequency state; randomize order, warm deliberately, and require correctness before accepting time.

## 5. Weight loading, packing, and NUMA ownership

Loading stages storage reads, decompression/decoding, integrity, type conversion, packing, page allocation, and first-touch placement. Pipeline stages with bounded queues so fast storage cannot exhaust host memory while packing lags. Track source, verified, transformed, and packed bytes separately.

Packed weights are a cache keyed by source tensor hash, kernel-family layout version, precision parameters, and CPU feature set. A packed object stores byte extent, NUMA replicas or sharding, reference count, validation, and eviction state. Never reuse a packed object across incompatible scale ordering or kernel layout versions.

For sockets $s$ with local bandwidth $B_s$ and cross-socket traffic $Q_{remote}$, model execution time using per-socket demand plus remote-link contention, not aggregate bandwidth. Replicating weights removes remote reads but multiplies capacity and cold-start time; sharding saves capacity but requires placement-aware task scheduling.

## 6. Task graph, thread pools, and backpressure

A runtime task stores plan/node identity, input/output tensor generations, dependency count, priority/deadline, preferred NUMA node, required thread-team size, workspace, state, cancel epoch, timestamps, and error. Dependencies release a task only when all producer tensors are visible.

Use separate pools or policies for latency-sensitive request work, bulk model loading/packing, storage/network callbacks, and accelerator completion. A single unbounded pool can deadlock when tasks synchronously wait for callbacks queued behind them. Prefer asynchronous dependencies; if blocking is unavoidable, reserve callback capacity.

Parallel kernels need nested-parallelism rules. If the serving scheduler runs many requests and each kernel creates a full-core team, oversubscription destroys cache locality and tail latency. The runtime chooses inter-request versus intra-operator parallelism from batch/shape, core budget, and SLO, and passes an existing team rather than spawning threads per operator.

## 7. Heterogeneous device adapter

The CPU stack may partition a graph onto GPU/NPU devices. The adapter owns context and address-space identity, memory allocation/registration, executable loading, submission queue, dependencies/events, completion, faults, and reset epoch. Cross-device tensors carry an explicit placement and transfer event; a host pointer alone does not prove device visibility.

Choose copy, mapped/shared memory, or coherent access from bandwidth, latency, consistency, protection, and lifetime. Before publishing a device command, weights/inputs and descriptor bytes must be visible. Before CPU consumption, completion plus the declared memory-order boundary must hold. A device failure terminates or retries tasks according to idempotence and state ownership; it cannot silently rerun a stateful sampling or cache mutation.

## 8. Sizing and trade-offs

Runtime throughput obeys the slowest saturated layer. For arrival/work rate $\lambda_i$ and average service demand $D_{ij}$ at resource $j$, require $U_j=\sum_i\lambda_iD_{ij}$ below its safe target. Resources include storage/decompression, packing cores, NUMA memory, execution cores, device queues, and completion threads.

| Choice | Benefit | Cost or failure region |
|---|---|---|
| ahead-of-time plans | predictable startup and validation | shape/CPU variants multiply artifacts |
| JIT specialization | adapts to actual shapes/features | cold latency, code cache, security, reproducibility |
| aggressive fusion | fewer launches/temporaries | register/cache pressure and fewer reusable kernels |
| weight replication | local bandwidth | memory capacity and load/update time |
| global work stealing | load balance | NUMA/cache migration and tail interference |
| per-model pools | isolation | stranded cores and operational complexity |
| accelerator offload | throughput/density | submission, transfer, fault, and synchronization path |

## 9. Invariants, verification, and observability

Invariants: every optimized task is semantically attributable to source operations; tensor storage generations prevent stale reuse; aliases cannot overwrite live values; a task reaches one terminal state; cancellation epochs suppress late writes; kernel dispatch never violates metadata requirements; and packed weights match source/version/quantization.

Verify importer against framework outputs, each compiler pass against its input IR, kernels against reference operators across shapes/precisions/extrema, memory liveness with poison/red zones, task dependencies under arbitrary delay/cancellation, NUMA placement, and device faults. Expose graph/node/variant identity, task ready/run/wait, thread/NUMA placement, useful/packed/transferred bytes, workspace, compilation/cache time, fallback, CPU counters, and device submission/completion.

## 10. Minimum viable stack and growth path

1. Load one immutable model, interpret a small operator set, and compare every tensor.
2. Add typed tensor descriptors, a kernel registry, and one safe variant per operator.
3. Add static graph planning and liveness-based buffer reuse.
4. Add thread-team execution with bounded queues and NUMA placement.
5. Add weight packing and cached plan identities.
6. Add fusion/JIT/autotuning behind differential validation and fallbacks.
7. Add accelerator partitions and asynchronous device state.
8. Add dynamic shapes, multiple models, and concurrent serving only after state generations and observability are stable.

The stack is reconstructable when artifacts, IR, passes, plan, ABI, buffers, tasks, dispatch, placement, fallback, faults, tests, and metrics can all be specified without “the framework handles it.”

---

← [Performance and Research](03_Performance_Analysis_Profiling_and_Research_Frontiers.md) · next → [CPU Serving Runtime, Scheduler, and State](05_CPU_Serving_Runtime_Scheduler_and_State_Implementation_Blueprint.md)
