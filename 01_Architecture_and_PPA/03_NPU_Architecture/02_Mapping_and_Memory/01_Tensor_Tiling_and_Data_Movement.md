# Tensor Tiling and Data Movement — The Neural Processing Unit (NPU) Compiler–Memory Contract

> **First-time reader orientation:** A tensor is a multidimensional array. Tiling divides a large operation into blocks that fit the NPU’s local scratchpad memories. A legal tile must fit inputs, weights, partial sums, metadata, and buffering while keeping enough processing elements busy. Performance comes from reducing expensive movement and overlapping transfers with computation.

> **Abbreviation key — skim now and return as needed:** register file (RF); static random-access memory (SRAM); high-bandwidth memory (HBM); double data rate (DDR); network on chip (NoC);
> direct memory access (DMA); processing element (PE); multiply-accumulate (MAC); general matrix multiplication (GEMM); 8-bit integer (INT8);
> kibibyte (KiB); mebibyte (MiB).

> **Prerequisites:** [NPU Accelerators](../01_Compute_Dataflows/01_NPU_Accelerators.md), [Systolic, Spatial, and Vector Dataflows](../01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md), and [HBM](../../02_GPU_Architecture/02_Memory_System/02_HBM_and_Advanced_Memory_Systems.md).
> **Hands off to:** compiler mapping/search, scratchpad controllers, NoC scheduling, and host command execution. This page owns loop blocking and data-movement accounting.

---

## 0. Why this page exists

Moving an operand from HBM can cost orders of magnitude more energy than using it in a MAC. Tiling keeps a working subset in each on-chip level long enough to reuse it, but every larger tile consumes capacity and can reduce parallelism or double buffering.

```mermaid
flowchart LR
    Graph["operator graph"] --> Loops["loop nest + tensor shapes"]
    Loops --> Tile["tile / reorder / fuse / spatial map"]
    HBM["HBM / DDR"] --> Global["global buffer"]
    Global --> Cluster["cluster SRAM / NoC"]
    Cluster --> Local["PE-local RF / accumulators"]
    Local --> Compute["MAC / vector engines"]
    Tile --> HBM
    Tile --> Global
    Tile --> Cluster
    Tile --> Local
```

The mapping is correct only if capacity, bandwidth, ordering, boundary masks, and dependencies all hold simultaneously.

## Before the details: a tile is a capacity-and-reuse contract

A tensor operation is often too large to keep all operands on chip. Tiling chooses smaller ranges of loop indices whose input, weight, output, and metadata footprints fit a memory level. Those ranges determine how often each byte is reused before eviction and how much traffic must cross the next, more expensive boundary.

For matrix multiplication, a tile with dimensions $M_t$, $N_t$, and $K_t$ needs an input block $M_tK_t$, a weight block $K_tN_t$, and partial sums $M_tN_t$, multiplied by their data widths. Double buffering may require two copies of inputs or weights so transfer overlaps computation. Alignment, bank placement, halos, tails, and compressed metadata consume additional capacity. A mathematically fitting tile may still starve the array if its transfers cannot complete before the current tile finishes.

**Beginner checkpoint:** write a byte-accurate capacity equation and traffic equation before running a mapping search. The compiler search should explore legal, understood choices—not compensate for an undefined memory model.

## 1. Describe mapping at every level

For each loop dimension specify:

- tile factor at HBM→global, global→cluster, cluster→PE;
- temporal order within a level;
- spatial factor and array dimension;
- multicast/reduction behavior;
- buffer allocation and phase;
- padding/boundary handling;
- tensor layout and address stride.

The product of temporal and spatial factors across levels must cover the original loop extent (with legal padding). A mapping record should be machine-readable; prose like “use output stationary” cannot reproduce an experiment.

## 2. Capacity constraints for GEMM

For tile sizes $M_t,N_t,K_t$ and element bytes $b_A,b_B,b_C$, basic storage is

$$
S_A=M_tK_tb_A,\quad S_B=K_tN_tb_B,\quad S_C=M_tN_tb_C.
$$

With double-buffered inputs and one accumulator tile,

$$
2S_A+2S_B+S_C+S_{metadata}+S_{align}\le S_{buffer}.
$$

Triple buffering or overlapped output writes add more. Banking/padding can make physical allocation exceed logical bytes; compilers must use hardware allocation granularity.

For convolution, im2col may inflate activation traffic/storage. Direct mappings reuse sliding windows without materializing the expanded matrix but need more complex address generation.

## 3. Traffic accounting

For each tensor $o$ and hierarchy boundary $l$, count unique tile transfers $N_{o,l}$. Total bytes

$$
Q_l=\sum_oN_{o,l}\,size(o,l).
$$

Arithmetic intensity at boundary $l$ is

$$
I_l=\frac{operations}{Q_l}.
$$

There is a roofline at every level:

$$
P\le\min(P_{compute},BW_{HBM}I_{HBM},BW_{NoC}I_{NoC},BW_{SRAM}I_{SRAM}).
$$

A tile can be HBM-efficient and still be NoC/RF-bound because one operand is remulticast or partial sums spill.

## 4. Reuse factors

For GEMM tile:

- each $A[m,k]$ is reused across $N_t$ output columns;
- each $B[k,n]$ across $M_t$ rows;
- each $C[m,n]$ accumulates $K_t$ products.

Ideal tile-level reuse reduces input bytes per MAC as $M_t,N_t$ grow, but accumulator capacity grows $M_tN_t$. Increasing all dimensions is impossible under fixed SRAM, so mapping chooses which reuse is most valuable given operand precision and bandwidth.

Reuse only counts if lifetime stays within the buffer and scheduling actually avoids reload. Two loop orders with identical tile sizes can produce different traffic.

## 5. Double buffering and pipeline balance

Use ping/pong buffers:

1. DMA fills tile $i+1$;
2. compute consumes tile $i$;
3. output engine drains tile $i-1$.

Steady tile time is ideally

$$
T_{tile}=\max(T_{read},T_{compute},T_{write}),
$$

plus startup/drain. Buffer ownership needs explicit states (`FREE`, `FILLING`, `READY`, `COMPUTING`, `DRAINING`) and command/tile IDs. Backpressure must stop overwrite if compute or writeback slips.

DMA burst size, alignment, and outstanding count determine achieved bandwidth. A perfectly sized tile can still expose transfer if the DMA cannot fill enough channels/banks.

## 6. NoC mapping

Spatial placement decides communication:

- map consumers sharing weights along multicast-friendly rows/columns;
- keep reducers near producers;
- avoid routing all tiles through one global-buffer bank;
- align logical partitions with NoC bisection and memory-controller placement;
- reserve virtual networks for read, write, control, and reduction progress.

For operand with fanout $F$ and path tree edges $E_t$, network flit-hops scale with $E_t$, not merely one source read. Compare alternative mappings using total bytes × hops and peak link load, not aggregate bytes only.

Mapping several layers concurrently can improve pipeline utilization but creates interference and buffer fragmentation. Static time slots are predictable; dynamic arbitration is flexible.

## 7. Operator fusion

Fusion retains producer output on chip for a consumer, eliminating HBM/global-buffer round trips. Examples: convolution/GEMM + bias + activation; attention matmul + scaling + masking + softmax stages; residual add + normalization.

Fusion constraints:

- compatible tiling/layout/order;
- combined live-buffer capacity;
- producer/consumer rate balance;
- numerical semantics and reduction order;
- no required global synchronization between tiles;
- vector/special-function support near the array.

Saved traffic is roughly write of intermediate + later read, but fusion may force smaller tiles and reduce primary operand reuse. Evaluate net traffic and utilization.

## 8. Layout and transformations

Tensor layout maps logical indices to physical addresses. Choices include channel-first/last, blocked channels, matrix-major variants, swizzles, and sparse formats. Layout affects:

- contiguous DMA bursts and bank distribution;
- vector/matrix fragment loading;
- padding overhead;
- producer–consumer compatibility;
- host/framework conversions;
- compression regularity.

Layout conversion is an operator with bandwidth, scratch space, and latency. Avoid claiming a fast kernel while excluding its transforms from end-to-end results.

## 9. Mapping different operators

### Dense GEMM/convolution

Maximize weight/activation reuse and accumulator locality; dense arrays fit well.

### Depthwise/group convolution

Lower cross-channel reuse; partition array and avoid padding many idle PEs.

### Attention

Sequence-dependent GEMMs plus softmax/reduction; tiling may stream keys/values and retain score/output blocks to avoid materializing full attention matrices.

### Embedding/gather

Random memory, low arithmetic intensity; prioritize HBM channels, caching, compression, request coalescing, and vector gather rather than the matrix array.

### Elementwise/normalization

Bandwidth and reductions dominate; fuse with adjacent operators and use vector units.

One mapping heuristic cannot cover all. The architecture needs flexible buffer/network access and fallback engines.

## 10. Mapping search

Search space includes tile factors, loop permutations, spatial assignment, buffer bypass, dataflow, layout, fusion, and precision. Prune with:

- factor/divisibility and capacity constraints;
- bandwidth and port bounds;
- array shape utilization;
- dependency legality;
- NoC route/bisection constraints;
- numerical/fusion legality.

Cost function can combine cycles and energy:

$$
J=\alpha T+\sum_lQ_lE_l+\lambda A_{required}+penalties.
$$

Analytical search narrows candidates; cycle simulation validates contention, pipeline overlap, and tails. Calibrate energy/access and timing parameters to synthesized/macro data.

## 11. Runtime dynamism

Dynamic batch/sequence/sparsity and multi-tenant work make compile-time tiles imperfect. Options:

- precompile shape buckets and dispatch by runtime shape;
- tail kernels for boundaries;
- dynamic work queues across subarrays;
- elastic buffer allocation;
- runtime tile-size selection under memory pressure;
- preemption at tile/command boundaries.

Keep command descriptors self-describing: dimensions, strides, layouts, buffer addresses, precision, mapping ID, dependencies, and completion target.

## 12. Verification and counters

Invariants:

- tile coverage is complete with no duplicate/missing logical elements;
- capacity/port/bank constraints are respected after padding/granularity;
- buffer phase prevents overwrite/read-before-fill;
- dependencies and reductions preserve numerical contract;
- DMA bounds/permissions are checked;
- fused execution matches unfused reference within declared tolerance;
- command reset/cancel cannot mix tiles.

Counters:

- bytes/accesses per tensor per hierarchy level;
- reuse and arithmetic intensity per level;
- buffer occupancy/fragmentation and phase stalls;
- DMA burst/alignment/channel balance;
- NoC flit-hops and hot links;
- compute/data/write overlap;
- array shape/fill utilization;
- fusion bytes saved versus tile-size loss;
- layout conversion traffic.

## 13. Numbers to remember

- A mapping specifies loop factors/order/space at every memory and compute level.
- Double-buffer capacity includes two input tiles, accumulators, metadata, padding, and alignment.
- There is a roofline at HBM, NoC, SRAM, and register/local levels.
- Reuse counts only when scheduling keeps data resident through all uses.
- Fusion saves intermediate traffic but can reduce tile size/reuse.
- Layout conversions belong in end-to-end performance and energy.

## 14. Worked problems

### Problem 1 — buffer capacity

INT8 $A,B$ tiles use $M_t=N_t=128,K_t=64$; INT32 accumulators. Double-buffered inputs plus output need

$$
2(128\times64)+2(64\times128)+4(128\times128)=98{,}304\ \text{B}.
$$

Before metadata/alignment, it fits 128 KiB but not 96 KiB.

### Problem 2 — arithmetic intensity

That tile performs $2M_tN_tK_t=2{,}097{,}152$ operations. Reading each input tile once and writing INT32 output once moves $16{,}384+16{,}384+65{,}536=98{,}304$ B at that boundary, giving about 21.3 ops/B.

### Problem 3 — fusion trade

Fusion saves a 64 MiB intermediate write and read (128 MiB), but combined live state forces tiles that add 20 MiB of input reload. Net traffic saving is 108 MiB; then verify utilization and numerical/order effects.

## Cross-references

- **Compute/dataflow:** [NPU Accelerators](../01_Compute_Dataflows/01_NPU_Accelerators.md), [Systolic, Spatial, and Vector Dataflows](../01_Compute_Dataflows/02_Systolic_Spatial_and_Vector_Dataflows.md).
- **Compression/system:** [Sparsity, Quantization, and Compression](02_Sparsity_Quantization_and_Compression.md), [Host Interface, Memory Visibility, and Scheduling](../03_System_Integration/01_Host_Interface_Memory_Visibility_and_Scheduling.md).
- **Memory/modeling:** [HBM](../../02_GPU_Architecture/02_Memory_System/02_HBM_and_Advanced_Memory_Systems.md), [Accelerator and NPU Simulators](../04_Simulation/01_Accelerator_and_NPU_Simulators.md).

## References

1. A. Parashar et al., [“Timeloop: A Systematic Approach to DNN Accelerator Evaluation,”](https://ieeexplore.ieee.org/document/8695666/) ISPASS 2019.
2. Y. N. Wu et al., “Interstellar: Using Halide's Scheduling Language to Analyze DNN Accelerators,” ASPLOS 2018.
3. H. Kwon et al., “MAESTRO: A Data-Centric Approach to Understand Reuse, Performance, and Hardware Cost,” IEEE Micro 2020.
4. N. Jouppi et al., “In-Datacenter Performance Analysis of a Tensor Processing Unit,” ISCA 2017.
5. Y.-H. Chen et al., “Eyeriss v2: A Flexible Accelerator for Emerging DNNs,” JETCAS 2019.

---

**Navigation:** [Mapping and Memory index](00_Index.md) · [NPU index](../00_Index.md)
