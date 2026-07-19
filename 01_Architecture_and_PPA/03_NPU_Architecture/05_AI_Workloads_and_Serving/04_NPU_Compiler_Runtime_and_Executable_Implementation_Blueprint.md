# NPU Compiler, Runtime, and Executable Implementation Blueprint

> **Abbreviation key:** neural processing unit (NPU); artificial intelligence (AI); intermediate representation (IR); direct memory access (DMA); input-output memory management unit (IOMMU); central processing unit (CPU); graphics processing unit (GPU); application programming interface (API).

## 0. Purpose and design ideology

This chapter reconstructs an NPU software stack from framework import through a versioned executable and protected runtime submission. The design ideology is **move dynamic decisions into a compiler only when their assumptions are representable and validated; retain explicit runtime state for everything that remains dynamic**. A highly scheduled NPU reduces hardware control but increases compiler proof, artifact compatibility, and fallback obligations.

Read [AI Workload and Graph Mapping](01_AI_Workload_and_Graph_Mapping_to_NPUs.md) for mapping theory and [NPU Graph-Compiler and Array Blueprint](../06_Implementation_Blueprints/01_Graph_Compiler_and_Execution_Array_Implementation_Blueprint.md) for target command/array contracts.

## 1. Stack components and ownership

~~~mermaid
flowchart LR
    FW["framework model + parameters"] --> IMP["importer + semantic validator"]
    IMP --> IR["typed tensor/graph IR"]
    IR --> PART["capability partition + fallback"]
    PART --> OPT["precision / fusion / layout / tile / schedule"]
    OPT --> MEM["buffer lifetime + DMA/event plan"]
    MEM --> EXE["versioned NPU executable"]
    EXE --> RT["runtime: contexts / queues / memory / events"]
    RT --> NPU["firmware + device commands"]
    PART --> HOST["CPU/GPU fallback runtime"]
~~~

Importer owns semantic equivalence. Capability database owns legal target operations/limits. Compiler owns partition/transforms/schedule. Packager owns executable identity. Runtime owns live contexts, tensors, commands, dependencies, and faults. Driver/firmware owns queue consumption and device recovery. Fallback coordinator owns cross-device transfer and ordering.

## 2. Capability and IR schema

The capability database is versioned with hardware/firmware. For every operation it describes supported ranks/dimensions, layouts/strides/alignment, precisions and quantization schemes, sparsity, dynamic bounds, numerical behavior, fusion patterns, memory limits, and command features. Compiler and runtime reject mismatch rather than approximate.

IR values store symbolic/static shape, bounds/constraints, type/accumulator/quantization, logical and physical layout, memory space, alignment, alias/lifetime, mutability, placement/sharding, and source identity. Operations store semantics, side effects, control dependencies, numerical order, randomness, fallback eligibility, and quality attributes.

Dynamic shapes use bounded symbols and guards. Control flow can be statically unrolled, represented as target loops/predicates when supported, dispatched among compiled regions, or executed as host fallback. State which values cross boundaries and how device/host visibility is established.

## 3. Partition and fallback contract

Partitioning builds maximal or cost-effective NPU regions while considering transfers, conversions, synchronization, and fallback frequency—not merely operator support. A boundary record includes source/target devices, tensor schema/layout/precision, ownership, conversion, transfer method, dependency event, fault propagation, and expected cost.

Fallback correctness requires the CPU/GPU implementation to match declared numerical semantics and consume the same versioned state. A dynamic fallback in the middle of a stateful decode must either preserve/convert KV and random state or reject the plan; recomputing one operator is not enough when layouts or quantization differ.

## 4. Compiler pass pipeline

1. import and validate graph/tensor artifacts;
2. infer shapes/types/layouts/aliases and constant values;
3. canonicalize/decompose into the target semantic core;
4. partition NPU and fallback regions with boundary costs;
5. calibrate/select precision, quantization, sparsity, and conversions;
6. fuse under numerical, shape, liveness, and resource legality;
7. select layouts and lower operators to tensor/vector/reduction primitives;
8. choose tiles, dataflows, array partitioning, and multi-NPU sharding;
9. allocate scratchpad/workspace using buffer lifetimes and bank constraints;
10. schedule DMA, compute, vector, reduction, collective, and event dependencies;
11. validate address bounds/resource peaks/dependency acyclicity;
12. emit commands, constants/packed weights, relocation, profiles, diagnostics, and source map.

Each pass records before/after work, bytes by level, buffer peak, conversion/padding, array/vector utilization estimate, fallback nodes, and reason alternatives lost. A compiler without diagnostics prevents architecture research from separating mapping limits from hardware limits.

## 5. Executable package

An NPU executable contains format/target/firmware version, graph/plan identity, shape profiles and guards, command programs, weight/constant shards and layouts, relocation/symbol tables, buffer/lifetime/bank plan, DMA/event graph, precision/quantization metadata, parallel-group/topology requirements, workspace and queue bounds, entry points, fallback regions, profiling/source map, integrity/signature, and build manifest.

Executables are immutable. Runtime relocation fills approved addresses/context identifiers without rewriting semantics. Load-time validation checks versions, commands, resource bounds, memory ranges, dependencies, signatures, and model/quality identity before device admission.

Shape-profile dispatch chooses the narrowest compatible profile according to a policy. If none matches, options are bounded compilation, fallback, padding under legal semantics, or rejection. Record the chosen profile and guard result.

## 6. Runtime API and state

Public/runtime interfaces cover device discovery/capabilities, context creation, executable load/unload, memory allocation/import/export, tensor binding, command/request submission, events, synchronization, cancellation, faults, profiling, and reset.

A context record stores tenant/process/address space, device/partition, executable references, memory quota, command/event namespace, security state, reset epoch, health, and error. An invocation stores executable/profile/entry, tensor bindings and generations, dynamic dimensions, workspace, dependencies, command queue, completion, cancel epoch, timestamps, and terminal result.

Buffer state includes device/host/tier, virtual address, extent/alignment, tensor layout, owner, readiness/last-use events, reference count, IOMMU mapping, and zeroization. Submission validates bindings against executable tensor schemas and guards; publishes descriptor/producer index after memory visibility; and completes only after output visibility.

## 7. Engine cache, packing, and model residency

The compiler/engine cache key includes source artifact, compiler options, capability/firmware, shape profiles, precision/quality, topology, and fallback library versions. Cached plans are revalidated on load. Weight packing keys include tensor hash, layout/quantization/shard version, target, and compiler pass.

Model residency states Registered → Validated → Compiling/Loading → Relocating → Warming → Ready → Draining → Unloaded, with Failed/Rollback. Reference counts pin an executable and weights while invocations/live state use them. Multiple versions coexist during rollout within capacity.

## 8. Cost model and design trade-offs

The cost model predicts compute waves/fill/drain, array edge/padding, vector/reduction work, scratchpad bank/port demand, DMA and external bytes/concurrency, event/command overhead, communication, and critical path/overlap. Calibrate by target and shape regime.

| Choice | Benefit | Cost/failure region |
|---|---|---|
| coarse scheduled commands | control efficiency | inflexible dynamic work |
| many shape profiles | utilization | compile/storage/validation explosion |
| padding to a profile | reuse plans | wasted work and possible mask semantics |
| aggressive fusion | traffic/launch reduction | scratchpad pressure and fallback granularity |
| static memory plan | predictable capacity | poor dynamic concurrency |
| host fallback | coverage | transfer/synchronization/layout and tail |
| runtime compilation | adaptability | cold latency and fleet reproducibility |

## 9. Invariants, validation, and build path

Invariants: every source node belongs to one compiled/fallback region; boundary tensors preserve semantic schema; executable commands/resources match capability; buffer lifetimes/banks do not conflict illegally; DMA/event graph is acyclic or has defined loops; invocation bindings satisfy profiles; context epochs reject stale completions; each invocation terminates once.

Compiler/runtime observability preserves model/source-node → IR operation/pass → partition/region → executable entry/command → invocation/event identity. Emit pass diagnostics, guards/profile selection, fallbacks, operations/bytes, predicted cycles and resource peaks, load/relocation, queue/event/DMA timing, device counters, and first failing command/buffer generation. The build manifest and effective configuration make those traces reproducible.

Validate adjacent IRs, compiler passes, target command functional model, executable parser/security, runtime memory/dependencies, fallback equivalence, dynamic guards, faults/cancellation, and device output. Build importer plus one static dense region; capability/partition and fallback; typed executable; runtime memory/queue/events; quantization/fusion; shape profiles; compiler cache; multi-NPU; bounded dynamic compilation.

The stack is reconstructable when compiler legality and diagnostics, executable bytes/metadata, runtime objects, binding/fault behavior, and fallback state are specified without “the NPU compiler handles it.”

---

← [NPU Performance/Compiler Research](03_Performance_Compiler_Profiling_and_Research_Methodology.md) · next → [NPU Serving Engine, Scheduler, and State](05_NPU_Serving_Engine_Scheduler_and_State_Implementation_Blueprint.md)
