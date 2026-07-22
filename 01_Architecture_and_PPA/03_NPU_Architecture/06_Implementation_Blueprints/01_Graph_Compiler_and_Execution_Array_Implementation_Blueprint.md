# NPU Graph-Compiler and Execution-Array Implementation Blueprint

> **Abbreviation key:** neural processing unit (NPU); intermediate representation (IR); processing element (PE); direct memory access (DMA); central processing unit (CPU); graphics processing unit (GPU).

## 0. Purpose and design ideology

An NPU turns a tensor graph into spatial and temporal movement of tiles through arithmetic units and local memories. Its design ideology is **make regular work explicit and cheap, while defining an escape path for irregular work**. A dense array is efficient because it omits much of a CPU/GPU’s dynamic scheduling, but the omitted decisions reappear in the compiler, command format, buffer plan, and runtime.

The hardware and compiler jointly define one machine. If the compiler assumes a tile can remain resident but hardware silently evicts it, or hardware supports a dataflow the cost model never selects, peak operations are irrelevant.

## 1. Freeze the supported-model contract

For each operator family, specify:

- tensor ranks, dimension ranges, dynamic dimensions, layouts, and alignment;
- data/weight/accumulator/output precisions;
- rounding, saturation, overflow, denormal, NaN/infinity, and stochastic behavior;
- broadcasting, padding, dilation, grouping, transposition, and masking;
- sparsity representation and legality;
- deterministic/reproducible mode;
- maximum error or model-quality validation procedure;
- unsupported cases and the CPU/GPU fallback boundary.

Include matrix multiplication, convolution, elementwise/vector functions, reductions, normalization, activation, gather/scatter, embedding, attention, and collective/communication operations according to product scope. “Supports Transformer” is not a contract; sequence lengths, head dimensions, causal masks, key-value-cache layouts, and dynamic batching make distinct demands.

## 2. Compiler transformation pipeline

~~~mermaid
flowchart LR
    G["framework graph"] --> CAN["canonical tensor IR"]
    CAN --> LEG["legality + decomposition"]
    LEG --> NUM["precision / quantization"]
    NUM --> FUS["fusion + layout"]
    FUS --> TILE["tile + dataflow search"]
    TILE --> MEM["buffer lifetime + DMA schedule"]
    MEM --> CMD["target commands + descriptors"]
    CMD --> RUN["runtime executable"]
~~~

The intermediate representation (IR) must retain shape (static or symbolic), layout/stride, element type and quantization parameters, memory space, side effects, aliasing, numerical attributes, and source identity. Transformation passes state preconditions and postconditions. For example, fusion is legal only if intermediate observability, numerical reassociation, memory aliasing, and dynamic-shape behavior allow it.

Compilation stages:

1. import framework semantics and freeze constants;
2. canonicalize operations to a smaller algebra;
3. infer shapes/layouts and prove bounds for dynamic dimensions;
4. decompose unsupported operations or mark fallback regions;
5. choose precision/quantization and insert conversions;
6. fuse producer/consumer regions when lifetime and numerical rules permit;
7. enumerate tile shapes and dataflows under array/scratchpad/bandwidth limits;
8. allocate buffers, choose bank/layout, and schedule transfers and compute;
9. emit versioned commands/descriptors plus relocation and resource metadata;
10. validate generated work against an operation-level reference.

Every pass should produce a diagnostic: work count, bytes by memory level, live-buffer peak, predicted cycles, padding waste, array utilization, and reason an alternative was rejected. This makes the design ideology inspectable.

## 3. Target command and descriptor contract

Use a small, versioned command set rather than leaking raw internal wires to software. Typical commands include load/store tile, matrix/tensor compute, vector operation, reduction, synchronization/event, loop/repeat, predicate, collective, and completion/fault.

A compute descriptor needs operation, input/output buffer identifiers and offsets, logical shapes, physical tile shapes, strides/layouts, precision/quantization, accumulation mode, edge masks, selected dataflow, dependency events, produced event, and optional profiling tag. A transfer descriptor needs source/destination spaces, base addresses, multidimensional extents/strides, element/transaction size, conversion/compression, translation context, protection, dependency, completion, and error policy.

State validation occurs before execution: bounds, alignment, legal precision/shape, buffer ownership, dependency graph, and privilege. A malformed descriptor must fault deterministically without partially corrupting unrelated buffers.

## 4. Array and processing-element reconstruction

A processing element (PE) needs input operand registers or queues, multiply/functional unit, accumulator, routing inputs/outputs, valid/ready or scheduled phase state, precision mode, mask, and error/parity state. Specify whether accumulation remains local, shifts through the array, or writes a separate accumulator SRAM.

For a two-dimensional systolic array, data advances between neighboring PEs on scheduled cycles. Define injection skew, boundary behavior, pipeline latency, accumulation lifetime, drain, edge-tile masking, and what happens under a downstream stall. A globally scheduled array may not support arbitrary backpressure inside the wave; in that case input/output buffers must guarantee the full wave before launch.

For an output-stationary matrix tile, partial sums remain in PEs/accumulators while activation and weight fragments stream. Weight-stationary retains weights and moves activations/partial sums; row-stationary tries to exploit several reuse classes. Each reduces one traffic class while increasing another or constraining tile shapes.

The theoretical operations for matrix multiplication $[M,K]\times[K,N]$ are $2MKN$ under the multiply-plus-add convention. For an $R\times C$ array with one multiply-accumulate per PE per cycle, ideal compute cycles are approximately

$$C_{ideal}=\left\lceil\frac{M}{R}\right\rceil
\left\lceil\frac{N}{C}\right\rceil K,$$

but actual cycles add fill/drain, edge underutilization, data-loading gaps, reductions, and synchronization. A detailed model uses

$$C=\max(C_{compute},C_{activation},C_{weight},C_{output})+C_{startup}+C_{sync},$$

only when transfers overlap; otherwise non-overlapped phases add. The compiler must emit a schedule that makes the assumed overlap legal.

## 5. Local data-path layers

A practical compute tile contains:

- activation and weight input buffers with bank mapping;
- distribution networks or systolic neighbor links;
- PE array;
- accumulator storage with wider precision;
- vector/special-function path for activation, normalization, address/mask, and conversions;
- reduction network for sums/maxima;
- output packing/quantization;
- command sequencer and event interface.

These layers wire into one producer-to-consumer path: the sequencer decodes the commands the compiler emitted (Sections 2–3) and gates each stage against fill/drain events.

~~~mermaid
flowchart TB
    CMD["commands + descriptors<br/>(compiler output)"] --> SEQ["sequencer +<br/>event interface"]
    DMAI["input DMA"] --> AB["activation buffer<br/>(banked)"]
    DMAI --> WB["weight buffer<br/>(banked)"]
    AB --> DIST["distribution /<br/>systolic links"]
    WB --> DIST
    DIST --> PE["PE array"]
    PE --> ACC["accumulator SRAM<br/>(wide precision)"]
    ACC --> RED["reduction network<br/>(sums / maxima)"]
    RED --> VEC["vector / special-function<br/>(activation, norm)"]
    VEC --> PACK["output packing +<br/>quantization"]
    PACK --> DMAO["output DMA"]
    SEQ -. "gate launch" .-> PE
    SEQ -. "fill event" .-> DMAI
    SEQ -. "drain event" .-> DMAO
~~~

Specify ownership transfer. A DMA engine writes a buffer while it owns the producer phase; compute may read only after the arrival event. Compute owns accumulator regions until a drain event. Output DMA may consume only after finalization/quantization. Double buffers alternate epochs so producer and consumer never address the same physical bank region simultaneously unless multiport behavior is designed.

## 6. Numerical implementation contract

For every operation, define input decode, multiply precision, accumulation width/order, rounding point/mode, saturation, scaling/zero point, and output conversion. Integer quantization commonly represents real value as $x=s(q-z)$ with scale $s$, integer $q$, and zero point $z$. Multiplication requires combined scales and sufficient accumulator width; requantization must specify rounding and overflow.

Reduction order changes floating-point results because addition is not associative. A tree, linear accumulator, or tiled partial sum produces different bits. Decide whether the contract is bit exact, tolerance based, or deterministic for a fixed mapping. Training usually requires stricter gradient/accumulator behavior than inference.

Sparsity requires metadata decode, zero-skip scheduling, and load balance. Structured sparsity simplifies indexing and fixed throughput; unstructured sparsity saves arithmetic only if metadata and imbalance cost less than skipped work. State malformed-metadata behavior and whether output ordering is deterministic.

## 7. Tiling and capacity derivation

For matrix tile dimensions $M_t,N_t,K_t$ and element byte sizes $b_A,b_B,b_C$, a simple live-byte estimate is

$$S=M_tK_tb_A+K_tN_tb_B+M_tN_tb_C,$$

before double buffering, padding, metadata, and vector temporaries. With two input buffer phases, use roughly twice the input terms. Require $S$ below usable—not nominal—scratchpad capacity after reservations.

Traffic per tile depends on reuse. Arithmetic intensity is $2M_tN_tK_t$ operations divided by bytes transferred from the measured level. Increase a tile until capacity/banking limits, diminishing reuse, edge waste, or latency constraints dominate. A tile that maximizes arithmetic intensity can reduce serving performance if it monopolizes scratchpad and prevents concurrent requests.

Bank count and ports must serve simultaneous array consumption, DMA fill, vector access, and drain. If compute consumes $R_A$ activation bytes/cycle and $R_B$ weight bytes/cycle, the local fabric and banks must deliver their sum plus any concurrent output traffic. Average bandwidth is insufficient; a scheduled wave needs cycle-level guarantees.

## 8. Policy and architecture trade-offs

| Choice | Benefit | Cost/losing case |
|---|---|---|
| large fixed systolic array | dense efficiency | edge/irregular/dynamic underutilization |
| smaller replicated arrays | concurrency and shape fit | duplicated control/buffers and cross-array reduction |
| exposed scratchpad | predictable data reuse | compiler/runtime correctness burden |
| hardware cache | easier irregular access | tag/coherence energy and unpredictable replacement |
| rich command ISA | flexible mapping | decode/state/verification and compatibility |
| coarse commands | efficient control | less dynamic adaptability and harder fallbacks |
| wide accumulators | numerical safety | area, SRAM bandwidth, and conversion cost |
| specialized precision | density/energy | model portability and validation burden |

Design for a measured shape distribution, not one perfect matrix. Record utilization histograms across production tiles and keep a vector/irregular path sized for the fallback fraction.

## 9. Invariants and verification

Key invariants:

- each command reads only buffers whose producing event and ownership are complete;
- every output element receives exactly the required reduction terms under its mask;
- no accumulator is overwritten before drain/finalization;
- edge padding cannot contribute to valid output;
- descriptor bounds and address calculations cannot escape the assigned memory region;
- every accepted command completes, faults, or is canceled once;
- compiler-declared resource use equals hardware admission/allocation.

Verify PE arithmetic bit accurately, then array waves, edge tiles, dataflows, vector/reduction composition, and compiled operators. Randomize shapes, strides, precision, masks, bank layouts, stalls at legal boundaries, and errors. Differentially compare after every compiler pass so a numerical failure can be located before full execution.

## 10. Minimum viable implementation

1. One fixed-shape dense matrix command and one array, with software-preloaded local buffers.
2. Add edge masks and several precisions with bit-accurate references.
3. Add DMA descriptors and double-buffer events.
4. Add vector/reduction operations and fused sequences.
5. Add compiler tiling/layout search and dynamic dimension bounds.
6. Add attention/sparsity/irregular paths only with utilization and fallback evidence.
7. Add concurrent contexts or arrays after ownership, protection, and progress are proven.

The reconstruction is adequate when the graph legality, IR fields, descriptor format, PE/array state, dataflow schedule, numerical behavior, capacity model, and reference tests can all be written without guessing.

---

Next → [Scratchpad, DMA, Runtime, and Serving Blueprint](02_Scratchpad_DMA_Runtime_and_Serving_Implementation_Blueprint.md)
