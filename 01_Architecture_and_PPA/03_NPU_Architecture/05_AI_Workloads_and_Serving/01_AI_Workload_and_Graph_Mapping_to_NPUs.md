# AI Workload and Graph Mapping to Neural Processing Units — From Model Semantics to Device Schedule

> **First-time reader orientation:** A neural-network model is not submitted to a neural processing unit (NPU) as Python source or as a list of layer names. A framework captures tensor operations and their dependencies, a compiler proves and rewrites shapes/layouts/precision, a backend selects NPU implementations, and a scheduler assigns tiles, buffers, transfers, engines, and events. Performance depends on every transformation, so a research result must preserve evidence from the original model to the executed command stream.

> **Abbreviation key — skim now and return as needed:** artificial intelligence (AI); neural processing unit (NPU); central processing unit (CPU); graphics processing unit (GPU); intermediate representation (IR); machine learning (ML); Open Neural Network Exchange (ONNX); general matrix multiplication (GEMM); general matrix-vector multiplication (GEMV); multiply-accumulate (MAC); processing element (PE); convolutional neural network (CNN); large language model (LLM); key–value (KV) cache; mixture of experts (MoE); direct memory access (DMA); static random-access memory (SRAM); dynamic random-access memory (DRAM); high-bandwidth memory (HBM); network on chip (NoC); input/output (I/O); fast Fourier transform (FFT); single program, multiple data (SPMD); just in time (JIT); ahead of time (AOT).

> **Prerequisites:** [NPU Accelerators](../01_Compute_Dataflows/01_NPU_Accelerators.md), [Tensor Tiling and Data Movement](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md), and [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md).

---

## 0. The mapping stack has five contracts

The phrase “run the model on the NPU” hides five distinct contracts:

| Contract | Must preserve | May change |
|---|---|---|
| model semantics | outputs within an agreed numerical/quality tolerance | algebraically equivalent expression, fused boundaries |
| graph | data and control dependencies, side effects, state | constant folding, dead-code elimination, canonical operators |
| tensor representation | logical values and index meaning | physical layout, padding, packing, quantized encoding |
| device program | correct ordering, addresses, resource ownership | tiling, dataflow, buffering, engine assignment, overlap |
| deployment | model/version/request isolation and response semantics | batching, replica choice, sharding, scheduling |

An optimization is valid only if it respects every contract above it. A graph rewrite can be numerically legal but fail a model-quality target. A tile schedule can fit nominal SRAM yet be illegal because two engines need the same bank in the same cycle. A fast executable can violate serving semantics if a canceled request writes into a reused KV-cache page.

## 1. End-to-end compiler path

~~~mermaid
flowchart LR
    A["framework model + checkpoint"] --> B["capture / export"]
    B --> C["portable or framework IR"]
    C --> D["canonicalization + shape inference"]
    D --> E["precision / quantization + layout"]
    E --> F["fusion + graph partition"]
    F --> G["tensor-loop / kernel IR"]
    G --> H["tiling + dataflow + sharding"]
    H --> I["buffer assignment + DMA schedule"]
    I --> J["NPU commands / binary"]
    J --> K["runtime loads and launches"]
    F --> X["CPU/GPU/custom fallback"]
~~~

### 1.1 Capture and export

The input may be a framework program, a serialized graph, or a model package containing both computation and parameters. Capture must record more than operator names:

- concrete, symbolic, and bounded-dynamic dimensions;
- data types, quantization parameters, and layout constraints;
- constants versus mutable parameters versus per-request state;
- control flow and data-dependent shape operations;
- aliases and in-place updates;
- preprocessing, tokenization, decoding, and postprocessing boundaries;
- training/evaluation mode and stochastic behavior.

An exported ONNX graph or StableHLO program is an **IR**, not the device executable. It describes operations and tensors at a level that can still be repartitioned, fused, retiled, and lowered differently for different NPUs. StableHLO, for example, specifies high-level operation semantics and dataflow ordering; a backend still chooses layouts, buffers, instructions, and collectives.

### 1.2 Canonicalization and decomposition

Frontends frequently express the same mathematical operation in different ways. Canonicalization rewrites them into a backend's supported vocabulary. Examples include:

- `einsum` or batched matrix multiplication → normalized contraction dimensions;
- convolution → native convolution loops or an `im2col`/GEMM-like lowering;
- layer normalization → mean/variance or root-mean-square reduction plus scale/bias;
- attention → projections, score contraction, mask, softmax, value contraction, or one fused attention composite;
- high-level activation → polynomial/table primitives supported by a vector engine;
- training operators → forward, backward, gradient accumulation, and optimizer updates.

Decomposition increases coverage but can destroy performance if it materializes intermediates. A research artifact should show both the pre-decomposition graph and the post-fusion executable graph.

### 1.3 Shape analysis is an architectural decision

The compiler needs enough shape knowledge to decide array partitioning, tile sizes, bank allocation, and collective structure. Three common policies are:

| Policy | Benefit | Cost |
|---|---|---|
| fully static | aggressive tiling, layout, memory planning, and AOT compilation | one executable may cover only one shape |
| bounded dynamic | one executable covers values up to a declared bound | padding and conservative schedules waste capacity/work |
| bucketed specialization | a small executable set covers shape ranges | compilation cache, dispatch logic, and padding between buckets |

For a runtime dimension $n$ assigned to bucket $hat n\ge n$, padding efficiency is

$$
U_{pad}=\frac{\text{useful work at }n}{\text{executed work at }\hat n}.
$$

For a GEMM dimension directly proportional to work, this can be approximately $n/\hat n$; for attention score work it can approach $(n/\hat n)^2$ (the score matrix is $n\times n$, so its work scales as $n^2$). For example, a request with $n=300$ tokens dispatched to a $\hat n=512$ bucket wastes little on a projection GEMM ($U_{pad}\approx 300/512=59\%$) but far more on the attention score matrix ($U_{pad}\approx (300/512)^2=34\%$); the quadratic term is why sequence-length bucketing punishes attention hardest. Consequently, “supports dynamic sequence length” is incomplete without the shape-bound and masking strategy.

### 1.4 Layout is an index map, not cosmetic metadata

Intuitively, layout is the seating chart, not the guest list: the tensor holds the same values, but *which value sits at which physical address* decides whether one hardware access grabs a contiguous burst or scatters across banks. The same $X$ stored row-major versus channel-blocked is like one warehouse shelved by product versus by supplier — identical goods, but "fetch every unit of one product" is a single aisle in the first and a scavenger hunt in the second.

A logical tensor $X[i,j,k]$ may be stored in row-major order, channel-blocked form, head-major form, a tiled hardware-native layout, or a packed quantized representation. A layout map converts logical indices into a physical address:

$$
addr(i,j,k)=base+f(i,j,k,\text{strides},\text{tiles},\text{swizzle}).
$$

Layout determines burst length, SRAM bank, vector-lane alignment, systolic feed order, and collective partition. A layout conversion costs a read, a permutation, and a write unless fused into the producer or consumer. Layout propagation therefore solves a graph problem: select compatible layouts across many operators while minimizing conversions and padding.

## 2. Precision, quantization, and calibration enter before scheduling

Quantization changes operator semantics and physical traffic. For affine integer quantization,

$$
x\approx s_x(q_x-z_x),
$$

where $q_x$ is the stored integer, $s_x$ is a scale, and $z_x$ is a zero point. A dot product then needs a wider accumulator and correction terms if zero points are nonzero. For a length-$K$ dot product,

$$
\sum_{i=1}^{K} s_x(q_{x,i}-z_x)\,s_w(q_{w,i}-z_w)=s_xs_w\left(\sum_i q_{x,i}q_{w,i}-z_x\sum_i q_{w,i}-z_w\sum_i q_{x,i}+Kz_xz_w\right),
$$

so the three $z$-dependent sums are the corrections and vanish for symmetric ($z_x=z_w=0$) encoding. The compiler must decide:

- per-tensor, per-channel, per-group, or per-token scales;
- symmetric versus asymmetric representation;
- static calibration versus dynamically computed activation scales;
- where quantize, dequantize, and requantize operations occur;
- accumulator width, rounding, clipping, saturation, and exceptional-value behavior;
- which operators remain at higher precision for quality.

Calibration is a measurement process: run representative data, collect activation distributions, select encodings, and re-evaluate model quality. A hardware speedup claim is valid only for the precision configuration that meets the declared quality target. Lower-bit weights can reduce HBM traffic, but scale/metadata reads and dequantization throughput must be counted.

## 3. Fusion is a liveness and resource transformation

Suppose a projection is followed by bias, activation, and quantization. Without fusion, each boundary may materialize a tensor in HBM or shared SRAM. With fusion, the projection accumulator can feed vector operations and write only the final representation.

Intuitively, fusion is the difference between filing every rough draft in the cabinet between edits and keeping the page on your desk until it is final: the arithmetic is identical, but the round-trips to slow memory vanish. Each un-fused boundary pays a full write plus a full read of the intermediate; a fused chain reads the operands once, holds every intermediate in the accumulator or a register/SRAM tile, and writes only the result.

~~~mermaid
flowchart LR
    subgraph UNF["unfused - every boundary round-trips HBM"]
        direction LR
        U0["matmul"] -->|"write B"| UH1[("HBM")]
        UH1 -->|"read B"| U1["bias"]
        U1 -->|"write B"| UH2[("HBM")]
        UH2 -->|"read B"| U2["GELU"]
        U2 -->|"write B"| UH3[("HBM")]
        UH3 -->|"read B"| U3["quantize"]
        U3 -->|"write B/2"| UH4[("HBM int8")]
    end
    subgraph FUS["fused - one read, one write"]
        direction LR
        FH0[("HBM weights+input")] -->|"read"| F0["matmul"]
        F0 -->|"accumulator"| F1["bias"]
        F1 -->|"on-chip"| F2["GELU"]
        F2 -->|"on-chip"| F3["quantize"]
        F3 -->|"write B/2"| FH4[("HBM int8")]
    end
~~~

For intermediate tensor size $B$, eliminating one write/read pair saves roughly $2B$ bytes at that memory level.

Worked example (one projection epilogue). Take $T=4096$ tokens (say batch 8 × sequence 512, or a 4096-token prefill) and output width $N=4096$ with FP16 intermediates, so each intermediate tensor is $B=4096\times4096\times2=32\ \text{MiB}$. Un-fused, the three boundaries (matmul→bias, bias→GELU, GELU→quantize) each add one write and one read: $6B=192\ \text{MiB}$ of HBM traffic on top of the operand reads and the final int8 write. Fused, those intermediates never leave the chip, so only the int8 result ($B/2=16\ \text{MiB}$, identical either way) is stored. The $192\ \text{MiB}$ saved is roughly $0.1\ \text{ms}$ of pure HBM time at $2\ \text{TB/s}$ per block instance, and — just as important — it raises the epilogue's arithmetic intensity so the matmul stays compute-bound instead of stalling on memory.

The theoretical saving is not free. Fusion expands the live range of accumulators and inputs:

$$
S_{live}=\sum_i B_i D_i.
$$

where $B_i$ is object size and $D_i$ is the number of simultaneously live versions. Fusion can therefore reduce HBM bytes while increasing scratchpad pressure, bank conflicts, code size, or vector backlog. A compiler should split a fused group when resource pressure causes more stalls than the eliminated transfer saved.

**Attention fusion** is a stronger algorithmic transformation. Online softmax tiles $QK^T$, preserves running row maximum/sum/output state, and avoids materializing the full score/probability matrix. This changes the off-chip I/O complexity, not merely the number of launches. See [Transformer and Attention Engine Microarchitecture](../01_Compute_Dataflows/03_Transformer_and_Attention_Engine_Microarchitecture.md).

## 4. From an operator to a legal NPU schedule

A schedule must answer all of these questions:

1. Which loop dimensions are spatial across processing elements, cores, or devices?
2. Which dimensions are temporal, and in what order?
3. What stays in a processing element, accumulator SRAM, local scratchpad, shared SRAM, HBM, or host memory?
4. How large is each tile, including padding and metadata?
5. Which DMA transfer fills/drains each tile, and can it overlap compute?
6. Which bank and port services each access?
7. Which event proves a producer is complete and a buffer is safe to reuse?
8. Which vector/reduction phase consumes matrix output?
9. How are edge tiles, sparse tiles, and runtime-dependent work handled?
10. Where are cross-core/device collectives inserted?

The result is not just a kernel name. It is a resource-constrained event graph containing transfers, matrix/vector/reduction commands, barriers, collectives, and buffer lifetimes.

Concretely, that event graph is a software pipeline over double-buffered tiles: each tile flows fill → matrix → vector/reduction → drain, and while one buffer is being consumed the DMA engine fills the next, so transfer and compute overlap instead of serializing (question 5). A completion event on each drain proves the buffer is free before a later tile is allowed to refill it — the buffer's *lifetime*, not merely its size, is what memory allocation must respect (question 7), which is why two tiles can share buffer A only if their live ranges do not overlap.

~~~mermaid
flowchart LR
    subgraph BA["buffer A lifetime"]
        F0["fill 0"] --> M0["matmul 0"] --> V0["vector 0"] --> D0["drain 0"]
    end
    subgraph BB["buffer B lifetime"]
        F1["fill 1"] --> M1["matmul 1"] --> V1["vector 1"] --> D1["drain 1"]
    end
    M0 -. "compute 0 overlaps fill 1" .-> F1
    D0 -. "event: A free" .-> F2["fill 2 reuses A"]
    D1 -. "event: B free" .-> F3["fill 3 reuses B"]
~~~

## 5. Dense GEMM and batched GEMM mapping

For $C_{M\times N}=A_{M\times K}B_{K\times N}$, dense work is $MNK$ MACs. A $P\times Q$ systolic array often maps output rows and columns spatially and advances $K$ temporally.

For one $M_t\times N_t\times K_t$ tile, required storage before padding is approximately

$$
S_{tile}=b_A M_tK_t+b_BK_tN_t+b_C M_tN_t+S_{metadata}.
$$

Here $b_A$ and $b_B$ are bytes per stored input element. The final term uses $b_C$ bytes per output element only if completed outputs occupy the tile buffer; if wider live accumulators reside there, replace it with accumulator width $b_{acc}$. $S_{metadata}$ includes descriptors, scales, masks, and alignment overhead in bytes. This expression is for one resident version of each tile. Ping-pong buffering doubles only the buffers that actually overlap producer and consumer—for example, $2(b_AM_tK_t+b_BK_tN_t)$ when inputs are double-buffered while one accumulator remains live—rather than blindly doubling every term. The compiler searches tile sizes subject to:

- array edge utilization;
- fill/drain amortization;
- scratchpad capacity and bank bandwidth;
- HBM burst/coalescing constraints;
- accumulator capacity and reduction throughput;
- enough independent tiles to occupy cores.

For batched GEMM, the batch dimension can map across cores while $M,N$ map within a core. During LLM decode, the effective token batch often becomes $M$; continuous batching turns many GEMV-like projections into a GEMM, increasing weight reuse.

## 6. Convolution mapping

A direct convolution loop spans batch $N_b$, output height/width $P,Q$, output channel $K_o$, input channel $C_i$, and filter $R,S$. Useful MACs are

$$
N_{MAC}=N_bPQK_oC_iRS.
$$

Mapping choices include:

- **direct spatial convolution:** preserve sliding-window reuse in input line/window buffers;
- **implicit GEMM:** compute the logical `im2col` (“image to columns”) address on demand without storing the expanded matrix;
- **explicit `im2col`:** materialize sliding image/filter windows as matrix rows/columns for a simple GEMM backend, at the cost of potentially large expansion traffic;
- **Winograd/FFT-like transforms:** reduce multiplications for selected shapes while adding transforms, numerical concerns, and workspace.

Depthwise convolution has little cross-channel reduction and often underuses a wide matrix array; grouping multiple images/spatial positions or using a vector/spatial path can be superior. CNN mapping must count halo overlap between tiles (the input-border region adjacent tiles must both load because the filter window straddles the tile boundary), stride/dilation effects, and padding, not only nominal MACs.

## 7. Sparse GEMM and structured sparsity

Sparse execution adds a metadata/data alignment problem. Let dense elements be $N$, nonzero fraction $\rho$, value bytes $b_v$, and metadata bytes per nonzero $b_m$. Compressed bytes are

$$
B_{sparse}=\rho N(b_v+b_m)+B_{header}.
$$

Compression reduces storage only when $B_{sparse}<Nb_v$. Compute speedup additionally requires the NPU to skip absent MACs and keep processing elements balanced. The schedule must specify:

- sparse format and granularity;
- metadata decode rate;
- intersection/matching for one or both sparse operands;
- assignment by nonzero work rather than dense coordinate count;
- accumulation of out-of-order partial sums;
- dense fallback threshold.

Structured patterns are easier to decode and load-balance but constrain the model. Unstructured sparsity exposes more theoretical zeros but may spend the saved work on indices, routing, and imbalance. See [Dynamic Sparsity, MoE, and Irregular Execution](../01_Compute_Dataflows/04_Dynamic_Sparsity_MoE_and_Irregular_Execution.md).

## 8. Attention, prefill, and decode mapping

### 8.1 Prefill

For prompt length $S$, projections expose token-parallel GEMMs. Attention computes score and value products over many query/key blocks. A fused tiled mapping keeps query blocks and online-softmax state resident while streaming key/value blocks. The compiler chooses query/key tile sizes to balance:

- matrix-array shape utilization;
- scratchpad capacity for $Q,K,V$, score fragments, and running state;
- vector/reduction throughput for max, exponential, and sum;
- causal-mask skipped work;
- HBM traffic and KV-cache writes.

### 8.2 Decode

One new token per sequence gives small per-request matrix dimensions. Batching independent sequences along the token dimension raises array utilization and reuses weights, but each sequence has a different context length and KV page list. A decode schedule therefore combines a dense projection batch with a ragged attention traversal.

Grouped-query or multi-query attention shares K/V heads among query heads, reducing KV capacity and bytes. Paged KV storage reduces allocation fragmentation but turns logical token indices into page-table and block-offset lookups. The DMA/address engine must gather pages efficiently and prevent one long sequence from monopolizing the memory queue.

## 9. Embeddings, retrieval, and recommendation workloads

Embedding lookup is usually a gather, not a dense contraction. Performance depends on:

- table size and placement across HBM, host DRAM, pooled memory, or storage;
- index distribution, locality, and hot-row cache behavior;
- row width and burst utilization;
- duplicate-index coalescing;
- pooling/reduction and feature interaction;
- misses and network fetches in distributed tables.

The useful bandwidth for random rows can be much lower than peak sequential bandwidth. If a row uses $B_r$ bytes but each memory transaction transfers $B_t$ bytes, transaction efficiency is at most $B_r/B_t$ before bank/channel imbalance. Specialized gather/scatter engines help only if index decode, translation, and reduction keep pace.

For recommendation inference, dense multilayer-perceptron work may run on matrix arrays while sparse table lookup runs on a separate engine or host service. The end-to-end critical path includes their join, so reporting only dense-engine latency is misleading.

## 10. Mixture-of-experts mapping

A mixture-of-experts layer first computes router scores, selects $k$ experts per token, permutes tokens into expert batches, runs expert GEMMs, and restores token order. Across devices it commonly requires an all-to-all exchange.

If expert $e$ receives $n_e$ tokens and capacity is $C_e$, useful processed tokens are bounded by the overflow policy. Completion follows approximately

$$
T_{MoE}\ge T_{route}+T_{dispatch}+\max_e T_{expert}(n_e)+T_{combine}.
$$

The maximum, not the mean, captures stragglers. The compiler/runtime contract must decide whether expert placement is fixed, replicated, cached, or migrated; how token batches are padded; what happens on overflow; and whether communication overlaps expert compute. Real routing distributions are required for evaluation.

## 11. Dynamic shapes, control flow, and fallback

NPUs favor regular, statically scheduled work, while models may contain data-dependent branches, loops, variable-length tensors, sorting, nonzero extraction, and custom operators. Four implementation choices exist:

1. specialize and cache an executable for each shape/control class;
2. pad/mask into a bounded static program;
3. execute dynamic work on a programmable NPU vector/scalar path;
4. partition the region to CPU, GPU, or a custom accelerator.

Fallback is not free. A boundary may require synchronization, device-to-host transfer, layout conversion, precision conversion, cache maintenance, and a second runtime dispatch. For region $r$,

$$
T_r=T_{sync}+T_{transfer}+T_{convert}+T_{fallback\ compute}+T_{return}.
$$

Coverage should therefore be reported both by operator count and by reference execution time/bytes. A model with 99% of operators on the NPU can still be dominated by one unsupported synchronization-heavy operator.

## 12. Multi-NPU sharding enters the compiler graph

Large weights, activations, optimizer state, or KV caches may not fit one device. Common dimensions are:

- **data parallelism:** replicate weights, partition requests/batch;
- **tensor parallelism:** partition a contraction dimension and insert all-reduce, reduce-scatter, or all-gather;
- **pipeline parallelism:** place layer stages on different devices and stream microbatches;
- **sequence/context parallelism:** partition token positions and communicate attention/reduction state;
- **expert parallelism:** place different experts on different devices and route tokens all-to-all.

An SPMD compiler propagates tensor sharding through the graph and inserts resharding collectives. The critical question is not whether a tensor *can* be sharded, but whether the selected sharding minimizes the weighted sum of communication, padding, memory, and lost compute utilization on the real topology.

## 13. Worked lowering: one Transformer block

Start with a framework block containing normalization, Q/K/V projections, attention, output projection, residual, normalization, gated feed-forward network, and residual.

1. **Capture:** record batch/sequence bounds, hidden/head dimensions, weights, KV state, and causal mask.
2. **Canonicalize:** express projections and feed-forward layers as contractions; normalize reductions and broadcasts.
3. **Quantize:** pack eligible weights, attach scales, choose accumulator/output formats; preserve sensitive reductions if quality requires.
4. **Fuse:** combine normalization/quantization with projection prologues; use online-softmax attention; fuse projection/activation epilogues when scratchpad permits.
5. **Partition:** retain supported fused regions on the NPU; explicitly cost sampler or custom fallbacks.
6. **Tile:** assign matrix dimensions to arrays, token/head tiles to cores, and K/V tiles to DMA streams.
7. **Allocate:** place weights/KV/temporaries in HBM and scratchpad banks; double-buffer only where lifetime/capacity allow.
8. **Schedule:** create fill → matrix → vector/reduction → drain events, plus KV append and barriers.
9. **Shard:** partition weights/KV/request batch and insert collectives for the device mesh.
10. **Emit:** package device commands/binary with shape guards, memory plan, runtime metadata, and version requirements.

The research artifact should retain a table linking each original graph node to its fused executable region, chosen implementation, bytes, operations, and fallback status.

## 14. Training changes the mapping problem

Training adds activation storage, backward operators, gradient reductions, optimizer state, stochastic operations, and a stronger numerical-reproducibility contract. The backward pass may have different shapes and reuse than forward; optimizer updates are often bandwidth-bound vector work. Activation checkpointing trades stored bytes for recomputation. Distributed training additionally inserts gradient all-reduce or reduce-scatter/all-gather.

For parameter bytes $W$, gradient bytes $G$, optimizer bytes $O$, and live activation bytes $A$, a first capacity check is

$$
C_{device}\ge W+G+O+A+workspace+fragmentation.
$$

Inference-only NPU claims should not be generalized to training without modeling these structures and phases.

## 15. Mapping audit checklist

- Is the model/checkpoint/tokenizer/preprocessing revision frozen?
- Are shapes distributions and bounds recorded, not just maxima?
- Can every graph rewrite be traced to preserved semantics and quality evidence?
- Are layout conversions and padding visible in traffic/work counts?
- Does quantization include scales, correction, calibration, and accuracy?
- Is fusion legal for scratchpad capacity, bank ports, and live ranges?
- Are matrix, vector, reduction, DMA, and collective schedules all represented?
- Are sparse/MoE mappings driven by value and routing distributions?
- Are dynamic regions and fallbacks included in end-to-end time?
- Does multi-NPU sharding match the physical topology and memory capacity?
- Is the exact compiled executable or compiler report archived?

## 16. Worked mapping decision: partition a decode array or keep it monolithic?

Consider an LLM decode projection with effective token batch $M=32$, output dimension $N=4096$, and reduction dimension $K=4096$. Compare a monolithic $128\times128$ systolic array with four independently schedulable $32\times128$ partitions. For one 128-column output block, use the wavefront approximation

$$
C=K+M+128-2=4254\ \text{cycles}.
$$

The monolithic array performs $32\times128\times4096=16{,}777{,}216$ useful MACs but provisions $128\times128\times4254$ PE-cycles:

$$
U_{mono}\approx\frac{16{,}777{,}216}{128\times128\times4254}=24.1\%.
$$

One $32\times128$ partition has

$$
U_{part}\approx\frac{16{,}777{,}216}{32\times128\times4254}=96.3\%.
$$

Four independent decode batches can occupy four partitions concurrently. This supports partitionability **only** when request supply, weight multicast/bandwidth, accumulator/SRAM banks, and vector throughput sustain all four. At batch $M=128$ or during large prefill, the monolithic array also approaches the high shape utilization and avoids partition mux/control costs. The design decision therefore comes from the measured prefill/decode batch distribution and shared-resource counters, not the 4× arithmetic ratio alone.

## 17. Evidence chain, failure boundaries, and open questions

| Workload claim | Hardware/compiler mechanism | Governing theory | Observable evidence | Validation and failure boundary |
|---|---|---|---|---|
| decode batching improves projection efficiency | batch dimension mapped to array rows; weight tile reused | shape/fill utilization and weight-side operational intensity | executable shape bucket, issued/padded MACs, array-active mask, HBM weight bytes | sweep logical batch at fixed model/quality; fails when queue delay, padding, KV/vector traffic, or memory contention dominates |
| fused attention reduces HBM traffic | online softmax keeps scores/statistics on chip | liveness capacity plus hierarchical I/O bound | fusion report, scratchpad peak, score-tensor HBM bytes, vector/reduction backlog | compare materialized and fused variants; fails if scratchpad spills or vector/reduction throughput backpressures matrix compute |
| sparse/MoE execution saves work | metadata decode, work queues, token routing, expert placement | compressed-byte break-even and maximum-load completion | density per tile, metadata bytes/stalls, queue imbalance, expert token histogram, all-to-all trace | replay real values/routes; fails under unsupported patterns, skew, metadata bottlenecks, or dense fallback |
| a dynamic graph is “supported” | shape guards/buckets, programmable path, or graph partition | padding efficiency plus fallback boundary cost | selected executable, padded work, compile-cache events, fallback time/transfers | sweep shape/control distribution; fails when rare shapes recompile, overpad, or synchronize across devices |

Open research questions include joint compiler–microarchitecture optimization of partitionable arrays, shape bucketing under changing serving traffic, automatic fusion constrained by bank/liveness proofs, topology-aware sparse/MoE mapping, and portable NPU IRs that preserve enough layout/dataflow intent without freezing one implementation. A result should state the workload distribution over which its mapping wins and the first regime where it loses.

## References

1. OpenXLA, [StableHLO Specification](https://openxla.org/stablehlo/spec) — portable operation semantics and dataflow ordering.
2. OpenXLA, [XLA Operation Semantics](https://openxla.org/xla/operation_semantics) — collective, dynamic-dimension, gather/scatter, and other lowering semantics.
3. Y. Xu et al., “GSPMD: General and Scalable Parallelization for ML Computation Graphs,” arXiv 2021 — [paper](https://arxiv.org/abs/2105.04663).
4. T. Dao et al., “FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness,” NeurIPS 2022 — [proceedings](https://proceedings.neurips.cc/paper_files/paper/2022/hash/67d57c32e20fd0a7a302cb81d36e40d5-Abstract.html).
5. A. Parashar et al., “Timeloop: A Systematic Approach to DNN Accelerator Evaluation,” ISPASS 2019 — [paper](https://arxiv.org/abs/1811.04037).
6. N. P. Jouppi et al., “In-Datacenter Performance Analysis of a Tensor Processing Unit,” ISCA 2017 — [Google Research](https://research.google/pubs/in-datacenter-performance-analysis-of-a-tensor-processing-unit/).

## Cross-references

- [Systolic, Spatial, and Vector Dataflows](../01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md) explains the array-level schedule.
- [Tensor Tiling and Data Movement](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md) derives capacity, traffic, and overlap constraints.
- [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) owns numeric and metadata mechanisms.
- [End-to-End NPU Serving](02_End_to_End_AI_Inference_and_Serving_on_NPUs.md) begins where the compiled artifact is deployed.

---

← [AI Workloads and Serving index](00_Index.md) · next → [End-to-End AI Inference and Serving on NPUs](02_End_to_End_AI_Inference_and_Serving_on_NPUs.md)
