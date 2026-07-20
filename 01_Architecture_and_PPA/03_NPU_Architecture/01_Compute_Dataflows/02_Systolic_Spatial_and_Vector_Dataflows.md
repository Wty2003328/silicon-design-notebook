# Systolic, Spatial, and Vector Dataflows — Choosing What Moves

> **First-time reader orientation:** A dataflow maps loop iterations and operands onto compute units, time steps, local memories, and communication paths. In a systolic array, neighboring processing elements pass data rhythmically; in a vector unit, lanes operate together under explicit vector instructions. The useful comparison is reuse, movement, utilization, and control—not diagram shape.

> **Abbreviation key — skim now and return as needed:** graphics processing unit (GPU); neural processing unit (NPU); instruction set architecture (ISA); single instruction, multiple data (SIMD); simultaneous multithreading (SMT);
> static random-access memory (SRAM); high-bandwidth memory (HBM); first in, first out (FIFO); direct memory access (DMA); processing element (PE);
> multiply-accumulate (MAC); general matrix multiplication (GEMM); output stationary (OS); tera operations per second (TOPS); 8-bit integer (INT8).

> **Prerequisites:** [NPU Accelerators](01_NPU_Accelerators.md) (systolic/dataflow overview), [SMT, SIMD, and Vector Execution](../../01_CPU_Architecture/01_Core_Foundations/03_SMT_SIMD_and_Vector_Execution.md), and linear algebra notation.
> **Hands off to:** [Tensor Tiling and Data Movement](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md) for concrete schedules and [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) for irregular/narrow data.

---

## 0. Why this page exists

An NPU's arithmetic units are rarely the differentiator by themselves. The decisive choice is how tensor-loop iterations map onto processing elements (PEs), which operands remain stationary, which multicast, which accumulate locally, and how boundary traffic scales.

```mermaid
flowchart LR
    Loops["tensor loop nest\nM, N, K, batch, spatial"] --> Map["spatial / temporal mapping"]
    Map --> Array["PE array"]
    Weights["weights"] --> NetW["multicast / shift"]
    Acts["activations"] --> NetA["multicast / shift"]
    NetW --> Array
    NetA --> Array
    Array --> Reduce["local / tree / spatial reduction"]
    Reduce --> Output["output tiles"]
```

Dataflow is a reuse contract: each operand movement should feed enough operations to amortize its energy and bandwidth.

## Before the details: dataflow is a placement decision

Matrix multiplication can be written as nested loops over output rows, output columns, and the shared reduction dimension. Every multiply consumes one input activation and one weight and updates one partial sum. A dataflow decides which loop dimensions run across processing elements, which run over time, and which operand remains local long enough to be reused.

In an output-stationary mapping, a processing element keeps one or more partial sums while inputs and weights move. Weight-stationary keeps weights local and moves activations and partial sums. Row-stationary tries to exploit several reuse types. The names are shorthand; a complete mapping must state spatial dimensions, temporal order, buffering, multicast, and reduction paths. Array utilization then depends on whether tensor dimensions tile the physical shape without underfilled edges and long fill/drain periods.

**Beginner checkpoint:** count useful multiply-accumulate operations divided by available processing-element cycles. Then separate idle cycles caused by array shape, pipeline fill/drain, dependencies, imbalance, and missing data. Peak operations per second alone hides all five.

### Build a systolic schedule from a two-by-two multiplication

The term *systolic* can sound more mysterious than the circuit is. Derive it from the smallest nontrivial case. Let

$$
A=\begin{bmatrix}a_{00}&a_{01}\\a_{10}&a_{11}\end{bmatrix},\qquad
B=\begin{bmatrix}b_{00}&b_{01}\\b_{10}&b_{11}\end{bmatrix}.
$$

Four output-stationary PEs hold $C_{00}$ through $C_{11}$. $A$ values enter from the left and move right; $B$ values enter from the top and move down. Each PE contains two input registers, one partial-sum register, a MAC, valid/tag state, and two forwarding registers:

```mermaid
flowchart LR
  AIN["A in"] --> AR["A reg"]
  BIN["B in"] --> BR["B reg"]
  AR --> MAC["MAC<br/>multiply then add"]
  BR --> MAC
  AR -->|fwd A| FA["to right PE"]
  BR -->|fwd B| FB["to lower PE"]
  MAC --> PS["stationary ps = C"]
  PS -->|accumulate| MAC
```

If all four rows and columns begin simultaneously, the wrong operands meet. The boundary therefore *skews* row $i$ by $i$ cycles and column $j$ by $j$ cycles. This alignment rule creates the diagonal wavefront:

```mermaid
flowchart LR
  A0["a row 0"] --> P00["PE00<br/>C00"] --> P01["PE01<br/>C01"] --> FA0["fwd A"]
  A1["a row 1"] --> P10["PE10<br/>C10"] --> P11["PE11<br/>C11"] --> FA1["fwd A"]
  B0["b col 0"] --> P00
  B1["b col 1"] --> P01
  P00 --> P10
  P01 --> P11
  P10 --> FB0["fwd B"]
  P11 --> FB1["fwd B"]
```

The exact useful work is:

| Cycle | PE00 | PE01 | PE10 | PE11 | What changed |
|---:|---|---|---|---|---|
| 0 | $a_{00}b_{00}\rightarrow C_{00}$ | idle | idle | idle | leading corner enters |
| 1 | $a_{01}b_{10}\rightarrow C_{00}$ | $a_{00}b_{01}\rightarrow C_{01}$ | $a_{10}b_{00}\rightarrow C_{10}$ | idle | three PEs lie on the wavefront |
| 2 | idle | $a_{01}b_{11}\rightarrow C_{01}$ | $a_{11}b_{10}\rightarrow C_{10}$ | $a_{10}b_{01}\rightarrow C_{11}$ | trailing wave reaches the far PE |
| 3 | idle | idle | idle | $a_{11}b_{11}\rightarrow C_{11}$ | final result completes |

```wavedrom
{ "signal": [
  { "name": "clock", "wave": "p...." },
  { "name": "A row 0 in", "wave": "==xxx", "data": ["a00", "a01"] },
  { "name": "A row 1 in", "wave": "x==xx", "data": ["a10", "a11"] },
  { "name": "B col 0 in", "wave": "==xxx", "data": ["b00", "b10"] },
  { "name": "B col 1 in", "wave": "x==xx", "data": ["b01", "b11"] },
  { "name": "PE00 useful", "wave": "11xxx" },
  { "name": "PE01 useful", "wave": "x11xx" },
  { "name": "PE10 useful", "wave": "x11xx" },
  { "name": "PE11 useful", "wave": "xx11x" }
] }
```

This trace explains three properties that a block-level diagram hides. First, **fill and drain are geometric**, not a memory accident: only one PE can work at cycle 0 and only one at cycle 3. Second, valid bits must travel with data; a global “array enabled” signal would make idle PEs accumulate garbage. Third, a stall cannot be inserted into only one forwarding path. Either elastic buffers preserve operand pairing, or the affected wavefront region stops together.

### Evolve the dataflow by following the first bottleneck

The preceding array fixes repeated global reads, but it is not automatically the right answer for every operation. Each next feature responds to a measured failure:

| Observed failure on the baseline | Feature introduced | What must exist to enable it | New cost or failure mode |
|---|---|---|---|
| partial sums are repeatedly read and written | output-stationary accumulation | wide PE accumulator, clear/drain states, output ownership | long reductions occupy the accumulator; context switches become expensive |
| weights are reused by many batches but reloaded at every output | weight-stationary mode | PE weight register/file, preload command, weight-valid/version bits | poor benefit for batch-1 or frequently changing weights; moving wide partial sums costs energy |
| one source SRAM bank is read once per destination | multicast | destination mask, fanout tree, pipeline/credit state | wire capacitance and one blocked branch can throttle the tree |
| $M$ or $N$ is smaller than the physical array | independently partitionable subarrays | boundary muxes, separate tags/counters, merge/reduction path | extra routing and control lower dense-mode efficiency |
| normalization, activation, or gather leaves the array idle | vector lane path | vector register/SRAM ports, instruction sequencer, cross-lane reduction | centralized register/network energy; still inefficient for random memory latency |
| operand DMA arrives late | double buffering and access–execute decoupling | two buffer phases, ready/release events, queues and credits | SRAM capacity and state-space double; bad reservation can deadlock |

Now replay the $2\times2$ example in a vector engine. Two lanes can compute a row of $C$ together: broadcast $a_{00}$ while loading $(b_{00},b_{01})$, then broadcast $a_{01}$ with $(b_{10},b_{11})$. This takes two vector MAC steps for one output row and repeats for row 1. There is no diagonal fill/drain and changing the vector length is easy, but each step reads a vector register and traverses a broadcast/cross-lane network. The systolic version pays four ramp cycles yet forwards operands through tiny local registers. Consequently, vectors win for short, irregular, or rapidly changing shapes; the systolic array wins when a long regular stream amortizes the ramp and local movement. A heterogeneous NPU exists because neither losing case can be repaired by scheduling alone.

### Trace-driven control and verification

A real tile command should expose at least `(tile_id, m_valid, n_valid, k_count, dataflow, precision, input_bank, weight_bank, output_bank)`. At launch, the controller clears or restores the selected accumulators, initializes edge-skew counters, and attaches `tile_id` to the first valids. Each hop advances data, valid, reduction index, and tag together. At the trailing edge, a PE asserts output valid only after observing exactly `k_count` useful pairs. Boundary masks allow the same schedule to run a partial edge tile without committing padded results.

The cycle trace gives direct verification properties rather than generic advice:

- at PE$(i,j)$, the first legal pair cannot arrive before cycle $i+j$ after launch;
- whenever a PE performs a MAC, its $A$ and $B$ tags, tile IDs, and reduction index must match;
- each unmasked PE performs exactly `k_count` MACs before one output-valid event;
- a stalled output cannot overwrite its accumulator, and backpressure must not separate a forwarded datum from its valid/tag;
- masked rows/columns may switch internally but cannot update architectural output;
- a buffer phase may be released only after the last wavefront that references it has passed.

Counters should mirror the same states: useful PE-cycles, leading/trailing ramp cycles, boundary-masked cycles, operand-starved cycles, backpressured cycles, multicast stalls, subarray occupancy, and vector-fallback cycles. With those counters, the designer can decide whether the next improvement belongs in array shape, buffering, network, or compiler mapping instead of treating every low-utilization kernel as “not enough TOPS.”

## 1. Start from the loop nest

Matrix multiplication $C_{mn}=\sum_k A_{mk}B_{kn}$ is the canonical three-loop computation:

```text
for m in M
  for n in N
    for k in K
      C[m,n] += A[m,k] * B[k,n]
```

Convolution expands this with batch, output/input channel, and spatial/filter loops. Any loop can be:

- **spatially unrolled** across PEs/lanes;
- **temporally iterated** on one PE;
- **tiled** at a memory hierarchy level;
- **reordered** subject to dependencies;
- **reduced** locally, across an interconnect, or in memory.

A mapping is the assignment of every loop dimension at every spatial/memory level. Labels like “weight stationary” summarize one consequence; they do not fully specify the mapping.

## 2. Systolic execution

In a systolic array, operands move rhythmically between neighboring PEs while partial results accumulate. Local communication replaces repeated global-buffer reads.

For an $R\times C$ array multiplying an $M\times K$ tile by $K\times N$:

- array rows often map $M$;
- columns map $N$;
- time maps $K$;
- activations shift across one dimension;
- weights shift across the other;
- outputs/partial sums remain or drain.

Ideal full-tile latency for a basic 2D wavefront is approximately

$$
T\approx K+R+C-2
$$

cycles for fill, compute wave, and drain, depending on exact interface/pipeline. Utilization is useful MACs divided by $RC\times T$ MAC slots.

Small or skinny matrices underfill dimensions. Mapping several independent tiles/batches across subarrays improves utilization but requires partitionable distribution/reduction and buffer capacity.

## 3. Stationary dataflows

### Output stationary (OS)

Each PE holds one or several partial sums while activations/weights stream. It minimizes high-precision partial-sum movement, attractive because accumulators are wider than operands. It needs efficient operand multicast/shift and final output drain.

### Weight stationary (WS)

Weights remain in PE-local storage; activations stream and partial sums move/reduce. It benefits inference or batched reuse of a weight tile, but weight reloads hurt small batches or dynamic models.

### Input/activation stationary

Activations stay while weights/partial sums move, useful when activation reuse dominates or weights change frequently.

### Row stationary / hybrid

Maps a convolution row/primitive to PEs to exploit weight, activation, and partial-sum reuse together. It is a family of mappings rather than one fixed layout.

Traffic energy is

$$
E_{data}=\sum_{operand\ o}\sum_{level\ l}N_{o,l}E_{access,l}.
$$

The best stationary choice minimizes high-level accesses under actual tensor shape and buffer constraints, not a universal rule.

## 4. Spatial arrays versus vector engines

| Dimension | Spatial/systolic array | Vector/SIMD engine |
|---|---|---|
| control | distributed/static schedule | centralized instruction issue |
| communication | PE-neighbor/multicast/reduction network | register file + cross-lane network |
| regular dense efficiency | very high | high but more operand movement/control |
| irregular ops/control | underutilized or requires fallback | more flexible |
| precision/shape changes | may fragment array | configurable lane/vector operations |
| compiler burden | spatial mapping and tile schedule | vectorization and instruction schedule |

Many NPUs combine matrix arrays with vector/scalar units for activation functions, normalization, reductions, data transforms, sparse gather/scatter, and control. The handoff between them can dominate if intermediates spill to a global buffer.

## 5. PE and local storage design

A PE may contain:

- one or several multipliers/MACs;
- accumulator registers or SRAM;
- operand registers/FIFOs;
- forwarding/multicast endpoints;
- zero/metadata logic;
- precision modes and lane grouping;
- local control and status.

Accumulator width must cover sum growth. For signed integer products of widths $b_a,b_w$ and $K$ terms, conservative magnitude needs roughly

$$
b_{acc}\gtrsim b_a+b_w+\lceil\log_2K\rceil
$$

bits plus sign/guard/saturation policy. Narrow input multipliers do not imply narrow accumulators or output networks.

Local storage amortizes network/global accesses but consumes area and can limit array tiling. Banking/ports must match operand arrival and result drain rates.

## 6. Distribution and reduction networks

Operand delivery patterns:

- unicast for unique elements;
- row/column multicast for reused operands;
- shift/systolic nearest-neighbor forwarding;
- broadcast trees;
- scatter/gather for sparse/indexed data.

Partial sums reduce through:

- PE-local temporal accumulation;
- linear chain;
- tree network ($O(\log P)$ depth);
- hierarchical cluster reduction;
- global-buffer read-modify-write.

The network must sustain required fanout without replicating source reads. If one activation feeds $F$ PEs, source bandwidth can be one word/cycle with multicast, but link/receiver energy still scales with distribution distance/fanout.

Multicast and reduction create backpressure: one slow branch can stall a lockstep tree unless buffered/decoupled. Systolic timing assumes balanced paths; physical skew/pipeline stages must preserve alignment.

## 7. Utilization losses

Total array efficiency decomposes:

$$
\eta=\eta_{shape}\eta_{fill/drain}\eta_{data}\eta_{sparse}\eta_{pipeline}\eta_{sync}.
$$

- **shape:** tensor dimensions do not fill array/subarray;
- **fill/drain:** wavefront startup and output drain;
- **data:** buffers/network/HBM cannot supply operands;
- **sparse:** irregular nonzeros unbalance PEs;
- **pipeline:** unsupported ops or dependencies bubble units;
- **sync:** tiles/subarrays wait at barriers/reductions.

For $M,N$ mapped to $R,C$, basic shape efficiency is

$$
\eta_{shape}=\frac{MN}{\lceil M/R\rceil R\cdot\lceil N/C\rceil C}.
$$

Array aspect ratio should follow workload shape distribution, not only one benchmark's square GEMMs. Reconfigurable subarrays trade mux/control/network cost for shape adaptability.

## 8. Precision reconfiguration

Bit-parallel units may pack multiple narrow operations into one wide multiplier/datapath; bit-serial units trade cycles for precision. Reconfiguration choices:

- fixed MAC lanes with several supported formats;
- split wide multiplier into independent narrow lanes;
- bit-serial/bit-sliced computation;
- shared exponent/block floating point;
- mixed input and accumulator precision.

Peak operations often count narrow packed modes. Compare energy, accumulator bandwidth, conversion overhead, and workload accuracy at the same numerical contract.

## 9. Pipeline and control

A spatial schedule is mostly static but still needs:

- tile/loop counters and address generators;
- buffer-ready/credit handshakes;
- multicast/reduction configuration;
- double-buffer phases;
- exception/error handling;
- synchronization between matrix/vector/DMA engines;
- predication for boundaries and sparsity.

Decoupled access/execute lets DMA fill the next tile while the array computes. Queues absorb latency variation; tags/epochs prevent a reset or fault from mixing tiles.

## 10. Mapping irregular operators

Depthwise convolution, small-batch GEMV, attention softmax, normalization, embeddings, and sparse expert routing stress different units.

- GEMV has low weight reuse and skinny dimensions; bandwidth/vector execution may dominate.
- Depthwise convolution lacks cross-channel reduction; large matrix arrays underfill.
- Attention includes GEMMs plus softmax/reduction and changing sequence dimensions.
- Embeddings are random memory gathers with little arithmetic.
- Elementwise ops need vector bandwidth and fusion to avoid round trips.

An NPU needs either flexible compute paths or graph fusion that keeps unsupported work near the array. Peak dense GEMM utilization is not a proxy for end-to-end model performance.

## 11. Verification and counters

Invariants:

- each logical loop iteration maps to exactly one required operation;
- reduction includes each term exactly once and uses defined order/rounding;
- boundary masks suppress padded work from architectural outputs;
- buffer phase does not overwrite live data;
- multicast delivers to exactly the configured destinations;
- precision/overflow/saturation behavior matches the ISA/compiler contract;
- reset/fault cannot mix partial sums from different commands.

Counters:

- active/total MAC slots by shape/fill/data/sparse cause;
- local/global/HBM accesses per operand;
- multicast fanout and network stalls;
- accumulator spills and reduction traffic;
- tile dimensions and subarray partition;
- matrix/vector/scalar engine utilization and handoff bytes;
- precision-mode residency and conversions.

## 12. Numbers to remember

- Dataflow maps every loop across space, time, and memory hierarchy; stationary labels are summaries.
- Systolic latency includes fill and drain, which dominate small tiles.
- Shape efficiency is useful tensor elements divided by padded array slots.
- Partial sums are wider than inputs and often worth keeping stationary.
- Multicast reduces source bandwidth but not all distribution energy.
- Dense peak TOPS does not predict irregular/elementwise/embedding performance.
- Fast convolution cuts multiplies orthogonally to dataflow: Winograd $F(m\times m,r\times r)$ needs $(m+r-1)^2$ vs $m^2r^2$ direct — $2.25\times$ for $F(2,3)$, $4\times$ for $F(4,3)$, ceiling $r^2$ — while $1\times1$/depthwise stay direct and large filters use FFT.

## 13. Worked problems

### Problem 1 — systolic utilization

A $64\times64$ array computes a $48\times96$ output tile with $K=128$. Two column passes are needed. Useful MACs are $48\times96\times128$. Available MAC slots ignoring overlap are $2\times64\times64\times(128+64+64-2)$. Utilization is about

$$
\frac{589{,}824}{2\times4096\times254}\approx28.3\%.
$$

The simple estimate exposes both padding and fill/drain loss; batching/subarray partitioning can improve it.

### Problem 2 — accumulator width

Signed INT8×INT8 products summed across $K=1024$ need roughly $8+8+10=26$ bits plus sign/guard policy. A 32-bit accumulator is reasonable; accumulating directly into INT8 is not.

### Problem 3 — multicast value

One activation is reused across 64 PEs. Without multicast, global buffer performs 64 reads; with one read and a distribution tree, global accesses drop 64×, though tree links/receivers still switch. The energy saving depends on global-read versus network-hop energy.

## 14. Fast-convolution algorithms: Winograd and FFT

Every section so far reduced the *cost of a multiply-accumulate* — where operands live, how far they move, how densely the array is packed. Fast-convolution algorithms attack a different quantity: the *number* of multiplies a convolution needs at all. This lever is orthogonal to dataflow. A Winograd or fast Fourier transform (FFT) convolution still runs on the systolic or vector engine of §2 and §4; it merely presents fewer, differently shaped multiplications wrapped in cheap, add-only transforms. Because multiplier area and energy dominate a MAC lane (§5, §8), trading multiplies for additions is usually a win — until the transforms themselves dominate.

### 14.1 The minimal-filtering multiply count

Let $F(m,r)$ denote producing $m$ outputs of a finite-impulse-response (FIR) filter with $r$ taps. Directly, each output is an $r$-term dot product, so $F(m,r)$ costs $mr$ multiplies. Winograd's minimal-filtering theorem gives a smaller count:

$$
\mu\big(F(m,r)\big)=m+r-1 .
$$

*Derivation.* The $m$ outputs are $m$ consecutive coefficients of the product of the degree-$(r-1)$ filter polynomial $g(x)=\sum_j g_j x^j$ and a length-$(m+r-1)$ data window. That product has degree $m+r-2$, hence $m+r-1$ coefficients. By the Cook–Toom construction a polynomial product with $m+r-1$ result coefficients is obtained by evaluating both operands at $m+r-1$ distinct nodes $\beta_0,\dots,\beta_{m+r-2}$, multiplying the evaluations pointwise, and interpolating. Evaluation and interpolation are linear in the data — additions and multiplications by constant node powers — so only the $m+r-1$ pointwise products are data-dependent multiplies. Winograd's lower bound shows $m+r-1$ is optimal over a field with enough distinct elements. $\square$

In matrix form the 1-D algorithm is $Y=A^{T}\big[(Gg)\odot(B^{T}d)\big]$, where $B^{T}$ evaluates the data window, $G$ evaluates the filter, $\odot$ is the elementwise (Hadamard) product carrying the $m+r-1$ multiplies, and $A^{T}$ interpolates. Nesting the 1-D algorithm along both axes gives the 2-D layer form used in practice:

$$
U=GgG^{T},\quad V=B^{T}dB,\quad M=U\odot V,\quad Y=A^{T}MA .
$$

The Hadamard stage runs once per transform coordinate, and there are $(m+r-1)^2$ of them, so 2-D $F(m\times m,\,r\times r)$ costs $(m+r-1)^2$ multiplies against $m^2r^2$ for direct convolution ($m^2$ outputs $\times\,r^2$ taps). The reduction ratio is

$$
\rho=\frac{m^2r^2}{(m+r-1)^2}=\left(\frac{mr}{m+r-1}\right)^{2}.
$$

As $m\to\infty$ with $r$ fixed, $\rho\to r^2$ — the ceiling is the tap count (9 for a $3\times3$ filter, i.e. asymptotically one multiply per output) — but the transforms become numerically unusable long before that.

### 14.2 $F(2\times2,3\times3)$ in full

For the ubiquitous $3\times3$ filter with a $2\times2$ output tile, $m=2$, $r=3$, tile size $\alpha=m+r-1=4$, and the standard nodes $\{0,1,-1,\infty\}$ give

$$
B^{T}=\begin{bmatrix}1&0&-1&0\\0&1&1&0\\0&-1&1&0\\0&1&0&-1\end{bmatrix},\quad
G=\begin{bmatrix}1&0&0\\[2pt] \tfrac12&\tfrac12&\tfrac12\\[2pt] \tfrac12&-\tfrac12&\tfrac12\\[2pt] 0&0&1\end{bmatrix},\quad
A^{T}=\begin{bmatrix}1&1&1&0\\0&1&-1&-1\end{bmatrix}.
$$

The 1-D core computes, from $d=[d_0,d_1,d_2,d_3]$ and $g=[g_0,g_1,g_2]$,

$$
\begin{aligned}
&m_1=(d_0-d_2)\,g_0,\quad m_2=(d_1+d_2)\tfrac{g_0+g_1+g_2}{2},\\
&m_3=(d_2-d_1)\tfrac{g_0-g_1+g_2}{2},\quad m_4=(d_1-d_3)\,g_2,\\
&y_0=m_1+m_2+m_3,\qquad y_1=m_2-m_3-m_4 .
\end{aligned}
$$

Substituting collapses $y_0=d_0g_0+d_1g_1+d_2g_2$ and $y_1=d_1g_0+d_2g_1+d_3g_2$ exactly — **four** multiplies for two outputs where direct convolution needs six. The filter-domain terms $\tfrac{g_0+g_1+g_2}{2}$ etc. are the entries of $Gg$; for inference the weights are fixed, so $U=GgG^{T}$ is precomputed offline and costs nothing at run time. $B^{T}$ and $A^{T}$ hold only $\{0,\pm1\}$, so the input and inverse transforms are add/subtract only.

Counting the 2-D layer: $F(2\times2,3\times3)$ needs $(m+r-1)^2=16$ multiplies per output tile versus $m^2r^2=36$ direct — the promised $36/16=2.25\times$. Widening to $F(4\times4,3\times3)$ gives $6^2=36$ versus $16\cdot9=144$, a $4\times$ reduction.

### 14.3 Mapping the transform onto the array

The transforms wrap the array rather than replace it. $B^{T}dB$ and $A^{T}MA$ collapse to small adder trees at the array's input and output boundaries; the Hadamard stage is where the MAC array works. At each of the $(m+r-1)^2$ transform coordinates the elementwise product across $C$ input channels and $K$ output channels is an independent $[K\times C]\cdot[C\times\text{tiles}]$ matrix multiply, so Winograd turns one convolution into $(m+r-1)^2$ smaller GEMMs, each mapped by any stationary dataflow of §3.

```mermaid
flowchart LR
    d["input tile d\n(m+r-1) x (m+r-1)"] --> BT["input transform\nV = B^T d B (adds only)"]
    g["filter g (static weights)"] --> Gt["filter transform\nU = G g G^T (precomputed)"]
    BT --> HAD["per-coordinate GEMM\nM = U (Hadamard) V on MAC array"]
    Gt --> HAD
    HAD --> AT["inverse transform\nY = A^T M A (adds only)"]
    AT --> y["output tile Y\nm x m"]
```

The overhead amortizes. The input transform runs once per input channel per tile (independent of $K$); the inverse runs once per output channel per tile (independent of $C$); only the Hadamard multiplies scale with $K\cdot C$. Take $K=C=64$ for one output tile: direct convolution needs $K\cdot(2\times2)\cdot(3\times3)\cdot C=36\cdot4096=147{,}456$ multiplies, Winograd needs $16\cdot K\cdot C=65{,}536$ — the same $2.25\times$, eliminating $81{,}920$ multiplies. The transforms add only additions (on the order of 32 per input tile and 24 per output tile in the Lavin–Gray factoring), a few thousand add-only operations here — negligible beside the eliminated multiplies, since a multiply costs several times an add in a MAC lane. When $C$ and $K$ collapse to $1$ (depthwise, §10), that amortization vanishes and the transforms dominate.

### 14.4 FFT for large filters

Winograd's small integer transforms are ideal for $3\times3$ but degrade as the tile grows. For large filters the fast Fourier transform (FFT) is the better route. The convolution theorem states that circular convolution is pointwise multiplication in the transform domain,

$$
x\circledast h=\mathrm{IDFT}\big(\mathrm{DFT}(x)\odot\mathrm{DFT}(h)\big),
$$

and an $N$-point FFT costs $O(N\log N)$ operations. Convolving a long signal with an $r$-tap filter by overlap-add — transform an $N$-sample block ($O(N\log N)$), multiply by the precomputed $H=\mathrm{DFT}(h)$ ($O(N)$), inverse-transform ($O(N\log N)$), and stitch — costs $O(\log N)$ multiplies per output sample, *independent of $r$*. Direct convolution is $O(r)$ per output, and Winograd's transform cost and error both grow with the tile $\alpha=m+r-1$, so FFT overtakes both once $r$ is large (roughly $r\gtrsim7$ for 2-D filters, and firmly for long 1-D filters). FFT needs complex arithmetic (or real-transform symmetry tricks), butterfly and bit-reversal datapaths, and a larger workspace, so for the dominant $3\times3$ it loses to Winograd's tiny transforms.

### 14.5 Trade-offs and when the simple path wins

- **Numerical inflation.** $B^{T}dB$ and especially the inverse $A^{T}MA$ take sums and differences that widen dynamic range; the transform matrices' condition number grows roughly geometrically with $\alpha$, amplifying rounding on the inverse. In FP16/INT8 this caps practical tiles at $F(2,3)$ and $F(4,3)$; $F(6\times6,3\times3)$ is rarely stable.
- **Transformed-weight and halo bandwidth.** For inference the stored transformed filter inflates from $r^2=9$ to $(m+r-1)^2=16$ values ($1.78\times$), and output-tile stride $m$ makes input tiles overlap by $r-1$, re-reading halos. A weight- or activation-bandwidth-bound layer can lose what the multiply count saved.
- **When direct/systolic wins.** A $1\times1$ convolution has $r=1$, so $\mu=m$ equals the direct count — Winograd is pure overhead, and the operation is already a GEMM for the systolic array of §2. Depthwise convolution has no channel reduction ($C=1$), so the Hadamard GEMMs are trivial and the transform overhead dominates. Both stay on the direct/systolic path.
- **When FFT wins.** Large kernels, as above.

Winograd is thus a targeted optimization for small dense spatial filters with real channel depth; it composes with the stationary dataflows of §3 and the tiling schedules of the mapping page, and its numeric widening is governed by the accumulator and format rules of §8 and the [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md) page.

## Cross-references

- **Overview/mapping:** [NPU Accelerators](01_NPU_Accelerators.md), [Tensor Tiling and Data Movement](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md).
- **Advanced workloads:** [Transformer and Attention Engine Microarchitecture](03_Transformer_and_Attention_Engine_Microarchitecture.md), [Dynamic Sparsity, Mixture of Experts, and Irregular Execution](04_Dynamic_Sparsity_MoE_and_Irregular_Execution.md).
- **Compression and scheduling:** [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md), [Decoupled Access/Execute and Scratchpad Scheduling](../02_Mapping_and_Memory/03_Decoupled_Access_Execute_and_Scratchpad_Scheduling.md).
- **Relatives/simulation:** [GPU Architecture](../../02_GPU_Architecture/01_Core_Architecture/01_GPU_Architecture.md), [Accelerator and NPU Simulators](../04_Simulation/01_Accelerator_and_NPU_Simulators.md).
- **Fast convolution:** Winograd/FFT (§14) map onto the systolic array (§2) and the [Tensor Tiling and Data Movement](../02_Mapping_and_Memory/01_Tensor_Tiling_and_Data_Movement.md) schedules; transform numerics follow §8, [Sparsity, Quantization, and Compression](../02_Mapping_and_Memory/02_Sparsity_Quantization_and_Compression.md), and [Floating Point](../../../00_Fundamentals/04_Floating_Point.md).

## References

1. N. Jouppi et al., [“In-Datacenter Performance Analysis of a Tensor Processing Unit,”](https://www.cs.cmu.edu/~18742/papers/Jouppi2017.pdf) ISCA 2017.
2. Y.-H. Chen et al., “Eyeriss: An Energy-Efficient Reconfigurable Accelerator for Deep Convolutional Neural Networks,” JSSC 2017.
3. H. Kung, “Why Systolic Architectures?” *Computer*, 1982.
4. A. Parashar et al., “Timeloop: A Systematic Approach to DNN Accelerator Evaluation,” ISPASS 2019.
5. H. Kwon et al., “MAESTRO: A Data-Centric Approach to Understand Reuse, Performance, and Hardware Cost of DNN Mappings,” IEEE Micro 2020.
6. S. Winograd, “Arithmetic Complexity of Computations,” SIAM, 1980.
7. A. Lavin and S. Gray, “Fast Algorithms for Convolutional Neural Networks,” CVPR 2016.

---

**Navigation:** [Compute Dataflows index](00_Index.md) · [NPU index](../00_Index.md)
