# GPU Software, Simulator, Verification, and Bring-up Blueprint

> **Abbreviation key:** graphics processing unit (GPU); intermediate representation (IR); instruction set architecture (ISA); streaming multiprocessor (SM); region of interest (ROI); high-bandwidth memory (HBM); network on chip (NoC); register-transfer level (RTL).

## 0. Why software is part of the implementation

A GPU does not execute a model or source-level kernel directly. A compiler selects instructions and resource usage; a runtime allocates memory and launches work; firmware creates contexts and queues; hardware consumes command descriptors; and a driver handles faults and completion. An implementation that omits these contracts can simulate impressive execution units but cannot run a real workload.

The design ideology is **make every transformation inspectable**. Preserve a stable identity from source operation to compiled instruction, command, warp instruction, memory transaction, and reported counter. This is also how simulation is validated: final runtime alone is insufficient if the intermediate work differs.

## 1. Complete software-to-hardware contract

~~~mermaid
flowchart LR
    SRC["framework / kernel source"] --> IR["compiler intermediate representation"]
    IR --> ISA["target instructions + metadata"]
    ISA --> BIN["code object"]
    BIN --> DRV["driver: context + virtual memory"]
    DRV --> RT["runtime: launch + events"]
    RT --> Q["hardware command queue"]
    Q --> DIST["grid/block distributor"]
    DIST --> SM["warps on SMs"]
    SM --> EVT["completion / fault / counters"]
    EVT --> RT
~~~

The code object must identify ISA/version, code/data sections, entry points, register use per thread, static shared memory, required dynamic shared memory, barrier/stack/local-memory requirements, launch bounds, relocation, constants, and optional debug/source maps. Hardware or firmware rejects unsupported versions/resources rather than interpreting them approximately.

A launch descriptor needs context/address-space identity, entry address, grid and block dimensions, argument-buffer address/size, dynamic shared-memory bytes, dependency/event list, completion destination, priority, and fault policy. Define ownership: software may modify a descriptor only before publishing its producer index; hardware may read it after an ordering barrier and owns it until completion.

Command queues need producer/consumer indices, wrap generations, doorbell ordering, descriptor alignment, error reporting, and reset recovery. The driver pins/maps pages, establishes permissions, flushes translation state as required, and ensures command/data visibility before ringing the doorbell. Completion visibility likewise requires an ordering point before software consumes results.

## 2. Compiler decisions hardware must support

Compilation performs parsing, target-independent optimization, parallel mapping, instruction selection, register allocation, scheduling, and binary emission. For AI kernels it also tiles tensors, selects tensor instructions, stages global-to-shared copies, vectorizes lanes, and inserts barriers/fences.

Hardware/software co-design requires a cost model with:

- instruction and pipeline latency/initiation interval;
- register and shared-memory allocation granularity;
- occupancy limits;
- bank/conflict and coalescing rules;
- cache and memory-transaction granularity;
- tensor shapes/precision constraints;
- asynchronous-copy capacity;
- launch and synchronization overhead.

An optimization is legal only if it preserves language/ISA memory and numerical semantics. It is profitable only if the saved work exceeds added resource pressure. For example, unrolling reduces branch/control overhead and exposes instruction-level parallelism, but raises register usage. If registers/thread cross an allocation boundary, resident warps can fall abruptly and make performance worse.

Store compiler decisions in metadata or diagnostic reports so simulation and profiling can correlate source lines with instruction mix, spills, occupancy, and memory traffic.

## 3. Exact simulator pipeline

Read [GPU Simulators](../04_Simulation/01_GPU_Simulators.md) for model types. A reconstructable simulator implements these stages:

1. **Input capture:** record source/binary hash, compiler version/options, launch dimensions, inputs, initial memory image, runtime calls, GPU configuration, and random seeds.
2. **Compilation or binary ingestion:** either run the real compiler and parse its target instruction stream, or explicitly define a virtual instruction set. Record register/shared-memory metadata and symbol mapping.
3. **Functional initialization:** create contexts, page tables, memory allocations, code objects, streams/events, command queues, and initial bytes exactly as the runtime would.
4. **Functional instruction execution:** form blocks/warps, evaluate opcodes and active masks, calculate addresses and values, update architectural state, and generate dynamic instruction/memory records. This establishes *what* happens.
5. **Timing admission:** allocate SM resources; only admitted warps enter scheduler slots. Apply instruction-cache, scoreboard, operand-collector, execution-pipeline, memory-queue, and barrier availability cycle by cycle.
6. **Memory timing:** translate addresses, coalesce lanes, access cache/MSHR/NoC/L2/HBM models, and schedule return events. Each event retains instruction/warp/transaction identity.
7. **Scale-up timing:** route peer/collective traffic through topology, link, credit, and synchronization models.
8. **Completion:** clear scoreboard/barrier/command state and publish runtime events only at the modeled visibility point.
9. **Aggregation:** derive cycles, instructions, utilization, bytes, hit rates, queue-delay distributions, energy, and service metrics from raw event timestamps and counts.
10. **Validation:** compare functional results, dynamic instruction counts, addresses, cache/partition traffic, and counters against hardware or a stronger reference.

The event queue stores timestamp, priority/phase, target resource, event type, payload identity, and cancellation generation. Same-cycle ordering must be deterministic: for example, whether completion makes a warp eligible in the same or next cycle changes throughput. Document evaluate/commit phases to avoid results depending on host-language iteration order.

## 4. Dynamic-work and timing separation

Three common models break different feedback:

- **execution-driven:** timing controls instruction progress and later addresses/control depend on modeled values; highest coupling and cost;
- **trace-driven:** replay a recorded instruction/address stream; fast, but altered cache/scheduling cannot change control or future addresses and traces may serialize away concurrency;
- **analytical:** use counts/distributions and formulas; fastest for exploration, but discards correlations, bursts, and synchronization unless explicitly modeled.

State which is used per subsystem. A functional frontend can generate instructions execution-driven while an analytical HBM model returns estimated delay. That hybrid is valid only within a declared boundary.

Warm-up must populate instruction/data caches, TLBs, predictor-like structures, memory row state, and steady occupancy as relevant. Region-of-interest (ROI) markers define which cycles/counts are reported. If sampling, preserve synchronization and memory state across skipped regions or quantify the error.

## 5. Result calculations

Define timestamps precisely:

$$IPC=\frac{N_{issued\ or\ completed\ instructions}}{C_{ROI}},\qquad
B=\frac{N_{transferred\ bytes}}{C_{ROI}/f}.
$$

State whether “instruction” means warp instruction or per-thread instruction, whether bytes are useful or physical, and which clock $f$ applies. Tensor utilization is useful array-active multiply-accumulates divided by available multiply-accumulate slots in the ROI; do not infer it from instruction issue alone.

Kernel latency runs from the selected launch/admission boundary to completion visibility. End-to-end serving includes host preparation, queueing, transfers, launch, kernels, collectives, and output according to the metric contract. Energy is the sum over events/structures plus leakage integrated over time; calibrate each event energy and avoid double-counting memory/NoC transfers.

Report counter conservation checks: issued memory transactions must equal completed + faulted + live-at-ROI-end; bytes entering a lossless network equal ejected plus buffered change; admitted blocks equal completed plus resident change. These catch simulator bugs before comparing performance.

## 6. Calibration and validation ladder

1. **Instruction semantics:** differential tests for every opcode, rounding mode, exceptional value, mask, and address space.
2. **Microarchitecture latency:** dependency chains for latency; independent operations for initiation interval; controlled bank conflicts and barriers.
3. **Memory hierarchy:** aligned/strided patterns, working-set sweeps, pointer chasing, partition mapping, TLB/page walks, HBM streams.
4. **Resource admission:** sweep registers/shared memory/block size and compare resident blocks/warps.
5. **Applications:** compare dynamic instruction mix, memory transactions, cache rates, occupancy, achieved clocks, and runtime.

Use multiple observables. If runtime matches but instruction count is 15% high and memory latency 15% low, the model is not validated. Hold out workloads not used for calibration and report error distribution, not one average.

## 7. Hardware verification plan

Cross-layer invariants connect the plan: compiler-declared resources equal hardware admission; a published launch descriptor is immutable until one terminal completion; every dynamic instruction and memory transaction has one live owner; simulator event conservation holds at ROI boundaries; masked/canceled work has no visible side effect; and a hardware counter with the same name and configuration measures the same event boundary as the simulator counter.

Verify layers in increasing composition:

- instruction/tensor numerical units against bit-accurate references;
- warp masks, divergence, scoreboards, barriers, and resource conservation;
- per-lane memory coalescing and shared-memory banking;
- cache/translation/atomics with reordered/backpressured responses;
- command processor, contexts, preemption, faults, and security isolation;
- multi-SM and multi-GPU progress, collectives, errors, and resets;
- compiler-generated random kernels compared to a software implementation.

Coverage crosses operation type with precision, mask pattern, dependency, resource pressure, fault, and preemption. Long random kernels need deterministic replay artifacts: binary, launch descriptors, memory image, seed, and first-divergence trace.

## 8. Bring-up and debug architecture

Provide command/warp trace triggers, per-SM halt/drain where safe, exception status with warp/lane/address, queue snapshots, register/shared-memory debug access under security control, NoC/L2/HBM counters, watchdogs, and feature disable switches.

Enable in gates:

1. clocks/reset/debug and firmware identity;
2. command fetch and a single arithmetic warp;
3. multiple warps, divergence, and barriers;
4. shared memory and global memory without caches;
5. caches, translation, faults, and atomics;
6. multiple SMs and resource admission;
7. tensor and asynchronous pipelines;
8. power states/preemption;
9. peer links and collectives;
10. full compiler/runtime and AI-serving workloads.

At each gate compare hardware trace/counters with the simulator using the exact binary and input. Keep conservative firmware modes—one SM, caches bypassed, reduced outstanding traffic, fixed clock—so the failing layer can be isolated.

## 9. Design trade-offs across layers

| Change | Hardware gain | Software/simulator obligation |
|---|---|---|
| larger tensor instruction | control amortization | compiler shape/legalization and edge-tile handling |
| more asynchronous copies | overlap | dependency groups, shared-memory lifetime, event-state model |
| fine preemption | service latency | context storage, safe points, fault/replay validation |
| relaxed memory/cache | throughput/area | correct fence insertion and scoped visibility tests |
| address hashing | partition balance | profiler mapping evidence and placement-aware cost model |
| specialized precision | energy/density | numerical contract, calibration, fallback, quality validation |

The build is reconstructable when the same contracts drive compiler legality, runtime descriptors, simulator state, RTL interfaces, performance counters, and bring-up tests. Divergent definitions across those layers are a design bug.

## 10. Completion checklist

- Can a source kernel be traced to exact dynamic warp instructions and memory transactions?
- Is every code-object and launch field consumed by a named owner and versioned?
- Does simulation explain how events contend cycle by cycle and how raw events become final metrics?
- Are trace-driven/analytical feedback losses explicit?
- Do compiler resource choices predict hardware admission and occupancy?
- Can hardware and simulator be compared at instruction, transaction, counter, and final-result boundaries?
- Does the bring-up plan isolate command, core, memory, tensor, power, and scale-up layers independently?

---

← [GPU Memory Blueprint](02_GPU_Memory_and_Scale_Up_Implementation_Blueprint.md) · [GPU Blueprint Index](00_Index.md)
