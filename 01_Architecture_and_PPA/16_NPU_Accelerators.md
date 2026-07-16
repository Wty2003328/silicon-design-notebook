# NPU Accelerators — Spatial Dataflow Hardware for Dense Linear Algebra

> **Prerequisites:** [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) (the MAC leaf this array is built from), [CPU_Architecture](03_CPU_Architecture.md) (the von-Neumann core this page deliberately departs from), [Performance_Modeling_and_DSE §10](01_Performance_Modeling_and_DSE.md) (the systolic cycle/utilization model this page motivates in hardware terms).
> **Hands off to:** [Full_Chip_Modeling §4](02_Full_Chip_Modeling.md) (PE→array→chip→pod power/thermal roll-up), [Accelerator_and_NPU_Simulators](06_Simulators/05_Accelerator_and_NPU_Simulators.md) (how a layer + mapping becomes a PPA number), and the companion AI-infra notebook's [Systolic_Arrays_and_Dataflow](../../silicon-to-serving/L2_Digital_Design_for_AI/03_Systolic_Arrays_and_Dataflow.md) / [Google_TPU](../../silicon-to-serving/L3_Microarchitecture/06_Google_TPU.md) for MXU-internal and vendor depth.

---

## 0. Why this page exists

A neural-processing unit (NPU) — the TPU-class (tensor processing unit) dataflow accelerator — is not a faster CPU; it is a *different shape of silicon*, built on one observation: a single kernel, dense matrix multiply (GEMM — general matrix multiply) and convolution, dominates deep-learning compute, and its structure is one a general-purpose core structurally cannot exploit. This page takes the **hardware-design lens** the rest of this notebook applies to the reorder buffer ([OoO_Execution §3](05_OoO_Execution.md)): *what must this silicon do and hold, and why does that force a spatial array of multiply-accumulate (MAC) cells fed by an explicitly-managed scratchpad, instead of a pipeline fed by caches?* The MXU-internal mechanics, the reuse arithmetic, and the vendor parts (TPU, Tenstorrent, Cerebras, Ascend) are the companion AI-infra notebook's territory; this page is the *why the hardware is shaped this way* those pages assume.

The single habit this page installs: **for a dense-linear-algebra accelerator, every architectural decision is a data-movement-energy decision first and a compute decision second** — because moving an operand costs orders of magnitude more than computing on it, and the whole machine is organized to move each operand as little as possible.

By the end you should be able to reason about an NPU quantitatively — size an array from a workload's shapes, read its roofline ridge *before* the silicon exists, choose a dataflow from an energy model rather than a cycle count, and predict where specialization stops paying — not recite a parts list.

---

## 1. Why a von-Neumann core is the wrong shape for dense GEMM

Start from the workload. A GEMM $C_{M\times N} = A_{M\times K}\,B_{K\times N}$ does $MNK$ MACs but touches only $MK+KN+MN$ operands, so its **operational (arithmetic) intensity** is

$$I = \frac{MNK}{MK+KN+MN}, \qquad \text{where } M,N,K \text{ are the GEMM dimensions;}$$

for a square $N\times N\times N$ GEMM, $I\approx N/3$ — reuse **grows without bound with problem size**, and it is *statically known* (no data-dependent control flow). Each element of $A$ feeds $N$ MACs, each of $B$ feeds $M$, each output accumulates $K$. That reuse is the resource an accelerator is built to capture; a machine that re-fetches operands per MAC throws it away.

A von-Neumann (temporal) core throws it away three times over:

- **Control/instruction overhead per MAC.** In a *temporal architecture* (CPUs, GPUs), one MAC is driven by an instruction: fetch, decode, rename, read 2–3 operands from the register file, execute, write one back. The *envelope* around the multiply-add — the control and operand delivery — dominates the energy: Horowitz's silicon measurements put the arithmetic itself below ~10% of what it costs to fetch/decode/schedule the instruction and read its operands from the register file, so the multiply-add pays a **10×+ overhead tax** every time (ISSCC 2014). A spatial array issues **one control decision for the whole array per step** and hardwires operands between cells, so that single control action is amortized over $D^2$ MACs — $65{,}536$ of them in a $256\times256$ array — and the per-MAC control energy falls by ~$D^2$. The tax does not shrink; it is *divided away*.
- **No direct operand reuse.** As the canonical survey puts it, a temporal architecture "use[s] a centralized control for a large number of ALUs [that] can only fetch data from the memory hierarchy and cannot communicate directly with each other." A *spatial architecture* has PEs (processing elements) that "form a processing chain so that they can pass data from one to another directly," each with "its own control logic and local memory, called a scratchpad or register file." That direct PE→PE path unlocks two reuse tiers — within a PE and between neighbors — that a temporal machine cannot express (Sze et al.).
- **MAC density.** A CPU/GPU spends most of its area on control (out-of-order logic, caches, schedulers); an NPU spends it on arithmetic. TPUv1 packs $256\times256 = 65{,}536$ INT8 MACs into one array at 700 MHz for **92 TOPS**, with a tiny control footprint (Jouppi et al., ISCA 2017).

The load-bearing point: **the accelerator's win is not a faster multiplier — it is deleting the per-MAC control and operand-delivery overhead by making the datapath *be* the schedule.** Because the loop nest is static, the "program" can be baked into wiring instead of re-decoded every cycle.

---

## 2. The systolic array as hardware

The canonical NPU datapath is the **systolic array** (Kung & Leiserson, 1978): a 2-D grid of identical PEs through which operands rhythmically pulse (*systole*), each PE doing one MAC per cycle on data arriving from its neighbors.

Apply the ROB-lens — derive what a PE must hold from the one job it does. A PE's job is: *multiply two operands arriving from adjacent PEs, accumulate, and forward operands onward in lock-step with the wavefront.* That forces it to contain exactly (and only):

1. a **multiplier + adder** — the MAC itself;
2. **one resident-operand register** — a held weight (weight-stationary) or an in-place accumulator for the partial sum (output-stationary): the tensor this PE keeps across cycles;
3. **edge/skew pipeline registers** so an operand handed to a neighbor stays time-aligned with the diagonal wavefront.

No instruction fetch, no register-file port arbitration, no bypass network, no branch predictor — **the PE's entire "program" is the fixed wiring to its four neighbors.** That is the structural reason an NPU PE is a fraction of the area and energy of a CPU issue slot.

```
weights wᵢⱼ preloaded and HELD in each PE (weight-stationary)
activations stream L→R; column partial sums accumulate downward

   x₀ → [w00]→[w01]→[w02] →        each PE, each cycle:
   x₁ → [w10]→[w11]→[w12] →          psum_down += w · x_in
   x₂ → [w20]→[w21]→[w22] →          x_out = x_in   (1-cycle skew)
           │      │      │
           ▼      ▼      ▼
          y₀     y₁     y₂          column sums drain out the bottom
```

Because a value streamed left→right is consumed by *every PE in the row*, and a partial sum passed top→bottom by *every PE in the column*, **each operand fetched once from SRAM feeds a whole row or column of MACs** — this is the systolic energy trick: TPUv1 "uses systolic execution to save energy by reducing reads and writes of the Unified Buffer" (Jouppi et al.). One SRAM read, $D$ MACs.

The cost of a rhythmic array is **fill and drain**. The intuition is geometric: the first result cannot emerge until an operand has propagated the $\sim D$ cells from the near edge to the far corner (**fill**), and after the last operand enters, the trailing wavefront needs $\sim D$ more cycles to flush through (**drain**). So a tile that does $K$ useful MAC-cycles occupies the array for $\approx K + 2D$ cycles — the $2D$ is pure overhead the useful work must amortize (full derivation: [Perf_Modeling §10.1](01_Performance_Modeling_and_DSE.md)). Utilization then factors cleanly into two losses:

$$U \;=\; \underbrace{\frac{M}{\lceil M/D\rceil D}\cdot\frac{N}{\lceil N/D\rceil D}}_{\text{edge-tile quantization}}\;\times\;\underbrace{\frac{K}{K+2D}}_{\text{fill/drain amortization}},$$

where $D$ = array side, $M,N,K$ = GEMM dimensions. The hardware consequence, stated plainly: **a systolic array is only efficient on matrices large relative to the array.** A skinny GEMM, a batch-1 decode step, or short-context attention leaves most PEs idle (edge quantization) or unamortized (fill/drain) — so array sizing is a *bet on the workload's shapes*, not a free "bigger is better." Put numbers on a $128\times128$ array ($D=128$): a fat training tile ($K=4096$) loses only $2D/(K{+}2D)\approx 6\%$ to fill/drain, so $U\approx 94\%$; a **batch-1 decode step** ($M=1$) collapses the edge factor to $M/D = 1/128$, so $U < 1\%$ — ~99% of the PEs are dark. The *same silicon* runs two orders of magnitude apart on training versus decode, from matrix shape alone; this is why inference serving fights so hard to *batch* requests, and why decode is memory-bound (§5). Modern TPUs use $128\times128$ MXUs, 2–4 per TensorCore (TPU v6e widens to $256\times256$); the smaller array trades peak density for higher utilization on realistic shapes (JAX scaling book; MXU-internal cadence is the companion [Systolic_Arrays_and_Dataflow](../../silicon-to-serving/L2_Digital_Design_for_AI/03_Systolic_Arrays_and_Dataflow.md)).

---

## 3. Dataflow — which operand stays resident, and why that is the energy lever

A **dataflow** is the choice of which tensor stays resident in the PEs while the others stream past. It is the accelerator's single most consequential decision, and the reason is *energy*, not cycles — the memory hierarchy has a steep energy gradient and the dataflow decides which tensor rides the cheap tier.

The gradient is the whole point. Normalized to the energy of one MAC, the cost of *reaching* an operand at each level is (Eyeriss, Chen et al., ISCA 2016, Table IV):

$$e_{\text{RF}} : e_{\text{PE-to-PE}} : e_{\text{global buffer}} : e_{\text{DRAM}} \;\approx\; 1 : 2 : 6 : 200,$$

where RF = the PE's local register file, PE-to-PE = an adjacent-PE hop, global buffer = the on-chip SRAM, DRAM/HBM = off-chip. Horowitz's independent silicon numbers agree: a DRAM access (1–2 nJ) is "a couple of orders-of-magnitude higher than the cost of an internal cache access or functional operation ($\sim$10 pJ)" (ISSCC 2014). As a ladder the whole thesis fits on one picture:

```
   cheap ┌──────────────────────────┐  energy per access (× one MAC)
         │  PE-local RF .......   1× │  ← the resident tensor lives here
         │  PE → PE hop .......   2× │  ← one SRAM read feeds a whole row/col
         │  on-chip buffer ....   6× │  ← the scratchpad; tiles refill here
    dear │  off-chip DRAM/HBM  200× │  ← every avoided fetch is paid here
         └──────────────────────────┘
   a dataflow's entire job: keep the *reused* tensor on the 1–2× rungs,
   and spend the 200× rung only on operands genuinely used once.
```

So an operand used $r$ times costs

$$E \approx \begin{cases} r\cdot e_{\text{DRAM}} & \text{re-fetched from DRAM each use} \\ e_{\text{DRAM}} + (r-1)\,e_{\text{RF}} & \text{fetched once, made resident} \end{cases}$$

— a $\sim$200:1 swing on a reused operand. **Choosing what stays resident is choosing which tensor pays the 1× rate and which pays the 200× rate.** That is the lever; the three canonical dataflows are three ways to pull it:

- **Weight-stationary (WS).** Each weight is loaded into a PE and held while a long stream of activations flows past — the weight is read from SRAM *once* and reused across the entire stream. Minimizes weight movement; wins when there are many activations per weight (large batch / large feature map). The TPU MXU dataflow.
- **Output-stationary (OS).** Each output's partial sum accumulates *in place* in a PE accumulator across the whole $K$-reduction, so partial sums — read-modify-write traffic, the most expensive kind — never spill to SRAM. Wins on long reductions (large $K$).
- **Row-stationary (RS).** Eyeriss keeps a 1-D convolution primitive (one filter row × one ifmap row → one psum row) resident and tiles the 2-D convolution across the array so weights, activations, *and* partial sums are all reused locally. It balances all three reuse types rather than maximizing one, which is why Eyeriss reported RS **1.4×–2.5× more energy-efficient than WS/OS/no-local-reuse on AlexNet** convolutional layers (a scoped, workload-specific result, not a global optimum).

The point to carry away: **dataflow changes cycles only modestly (through utilization) but changes energy substantially (through which hierarchy level absorbs the accesses)** — two mappings with identical MAC counts and near-identical cycle counts can differ 2–3× in energy. That is precisely why dataflow is chosen with an analytical *energy* model, not a cycle model; the mapping machinery (Timeloop/Accelergy, MAESTRO, SCALE-Sim) and the MXU-level dataflow mechanics (plus input-stationary and vendor variants) live in [Accelerator_and_NPU_Simulators §3–6](06_Simulators/05_Accelerator_and_NPU_Simulators.md) and the companion [Systolic_Arrays_and_Dataflow](../../silicon-to-serving/L2_Digital_Design_for_AI/03_Systolic_Arrays_and_Dataflow.md). Here it is enough that **dataflow = resident-tensor choice = which operand pays the cheap energy rate.**

---

## 4. Explicit on-chip memory and the mapping problem — a scratchpad, not a cache

An NPU's on-chip memory is a large **software-managed scratchpad** (TPUv1's 24 MiB Unified Buffer; tens of MB on modern parts) — a flat, addressed SRAM the compiler owns — **not** a hardware cache. That is a deliberate hardware decision, and the ROB-lens explains it: what must this memory do, and what does that *not* require?

A cache is *reactive*: it decides what to keep with run-time replacement heuristics (LRU), spends area on tag arrays and comparators, and has data-dependent hit/miss timing — all machinery for an access pattern that is *unknown until run time* ([Cache_Microarchitecture](07_Cache_Microarchitecture.md)). But an NPU's access pattern is **entirely known ahead of time** — the loop nest is static (§1). Given a schedule fixed in advance, every transistor spent on tags and LRU is wasted, and every run-time eviction decision is a chance to throw out a tile you provably need next. So the NPU deletes the cache and keeps two things instead: (1) the raw SRAM, addressed directly by the compiler, and (2) **DMA (direct-memory-access) engines** the compiler programs to stage the next tile while the array computes the current one. **The determinism that lets an NPU omit a branch predictor also lets it omit a cache — reactive hardware is replaced by a static schedule** (SRAM cell/array substrate: [Memory](09_Memory.md)).

The flip side is that the compiler now owns a hard **mapping/tiling problem**: choose tile sizes $(T_m, T_n, T_k)$ so a tile's operands fit the scratchpad, a loop order, and which loops unroll *spatially* onto the array versus *stream temporally*. This mapping **is** the accelerator's program (searched by the tools in [Simulators §2–5](06_Simulators/05_Accelerator_and_NPU_Simulators.md)). Two constraints bind it:

$$\text{(capacity)}\quad \text{footprint}(T_m,T_n,T_k)\le S_{\text{buf}}, \qquad \text{(overlap)}\quad t_{\text{tile}} = \max(t_{\text{compute}},\, t_{\text{DMA}}),$$

where $S_{\text{buf}}$ = scratchpad capacity, and the $\max()$ (not a sum) holds *only if the buffer is double-buffered* — big enough for two tiles, so DMA fill of tile B overlaps compute on tile A. If only one tile fits, movement and compute serialize back to $t_{\text{compute}}+t_{\text{DMA}}$ and throughput halves ([Full_Chip_Modeling §4.4](02_Full_Chip_Modeling.md)). Get the tiling wrong and a compute-bound layer becomes memory-bound — and, unlike an out-of-order core hiding a cache miss, **the fixed hardware cannot rescue a bad map at run time.**

---

## 5. The roofline view of an accelerator

The most useful first-order model of an accelerator is the **roofline** (Williams, Waterman & Patterson, CACM 2009). Attainable throughput is

$$P_{\text{attain}} = \min\bigl(\pi,\; \beta\cdot I\bigr), \qquad I^{*} = \frac{\pi}{\beta},$$

where $\pi$ = peak compute (TOPS/FLOPS the MAC array can sustain), $\beta$ = memory bandwidth (HBM — high-bandwidth memory), $I$ = operational intensity (useful ops per byte of off-chip traffic), and the **ridge point** $I^{*}$ is the intensity separating memory-bound (left, $\beta I$ binds) from compute-bound (right, $\pi$ binds).

Why this is *the* accelerator lens: an NPU is engineered for enormous $\pi$ (huge MAC density, §1) against a fixed $\beta$, so its ridge point is *high*. TPUv1 makes it concrete: 92 TOPS ($\approx 46$ TMAC/s) against 34 GB/s of DDR3 puts the ridge at $I^{*}\approx 1350$ MACs/byte (Jouppi et al.) — an operator must deliver *~1350 multiply-accumulates for every byte it touches off-chip* just to reach the compute roof, and most operators come nowhere close. Modern parts widen $\beta$ with HBM but widen $\pi$ faster, so the ridge stays in the hundreds-to-thousands. The consequence is stark: **an NPU is memory-bound on any operator whose reuse is low**, and the entire art of mapping (§4) is dragging operators rightward past the ridge by raising reuse (bigger tiles, resident tensors). A large-$M,N,K$ dense GEMM has $I\sim O(\text{dim})$ and sits compute-bound; a skinny GEMM, an elementwise op, or attention KV-cache reads at batch 1 have $I\sim O(1)$ and sit memory-bound — the array starves no matter how many MACs you built.

Because the silicon is heterogeneous (systolic array for GEMM + a SIMD **vector unit (VU)** for softmax/normalization/activation + HBM), the single roofline specializes into three per-operator bottleneck classes ([Perf_Modeling §10.3](01_Performance_Modeling_and_DSE.md)): **SA-bound** (at the compute roof), **VU-bound** (the vector unit is the critical path — common in attention's softmax), and **HBM-bound** (under the bandwidth roof). An operator-level simulator emits this classification directly ([NeuSim, Perf_Modeling §11](01_Performance_Modeling_and_DSE.md)); it is the accelerator's analogue of the CPI stack telling a CPU architect where the cycles go.

---

## 6. Scaling out — multi-die and pods, where the interconnect is the roofline

A single die cannot grow arbitrarily: the lithography **reticle limit** (~800 mm²) caps die area, yield falls with area, and a frontier model no longer fits in one chip's HBM. NPU scaling therefore goes *outward* — many dies wired into a fabric — and at that point **the interconnect, not the MAC array, sets performance.** Two levels:

- **Multi-die / chiplet (on-package).** Split what would be one over-reticle die into chiplets joined by a die-to-die link (UCIe, NVLink-C2C) over a silicon interposer — buys area past the reticle and better yield, at the cost of cross-die latency/energy and a bandwidth wall at the die boundary (packaging substrate: [IC_Packaging](../07_Manufacturing_and_Bringup/02_IC_Packaging.md); cross-die coherence: [ACE_and_CHI](12_ACE_and_CHI.md)).
- **Pod / cluster (across nodes).** Tens to thousands of chips over a dedicated **inter-chip interconnect (ICI)**. TPUs wire a 2-D or 3-D **torus** — chosen for bounded node degree, short diameter, and a natural map onto collectives — scaling to a 9,216-chip pod on TPU v7 (Ironwood). NVIDIA wires an **NVLink** fabric: 5th-gen NVLink gives 1.8 TB/s per GPU, an NVL72 domain ~1 PB/s aggregate.

Why the network dominates once you are here: a matmul sharded across chips needs **collectives** — chiefly **AllReduce** to sum partial products / gradients. Ring-AllReduce moves $\approx 2S$ bytes per chip *independent* of ring size $N$ (bandwidth term) but pays $2(N{-}1)$ hop latencies (derived in [Full_Chip_Modeling §4.4](02_Full_Chip_Modeling.md)), so at large $N$ or small payload $S$ the communication time approaches or exceeds per-step compute and scaling efficiency collapses — *unless* comm overlaps compute (AllReduce layer $L$ while computing layer $L{-}1$: the pod-scale twin of the on-chip double-buffering of §4). The hardware consequence: **a pod is a network with compute attached, and ICI bandwidth / torus bisection is as first-class a design parameter as the MAC count.** Pod topology, optical circuit switching, and the collectives/parallelism stack are the companion [Google_TPU](../../silicon-to-serving/L3_Microarchitecture/06_Google_TPU.md) page; this page's point is only that the reticle and HBM limits *force* the scale-out, and the interconnect becomes the binding roof (on-die operand fabrics and torus routing: [Network_on_Chip](13_Network_on_Chip.md)).

---

## 7. Trade-offs — the efficiency–flexibility ledger, and where dataflow hardware stops winning

The NPU buys its 10–100× perf/W over a CPU by *specializing*, and every element of that specialization is a bet that narrows the machine:

| Lever | Buys | Costs / risk |
|---|---|---|
| Spatial MAC array (vs temporal core) | amortized control, high MAC density (§1) | only dense, regular tensor ops; branches/irregularity fall to a scalar unit |
| Fixed dataflow (§3) | cheap resident-operand reuse | wrong for mismatched shapes (skinny GEMM, batch-1 decode) → low utilization |
| Scratchpad + DMA vs cache (§4) | no tag/LRU area, deterministic timing | compiler must map perfectly; a bad tiling can't be fixed at run time |
| Reduced precision (INT8/FP8/FP4) | 2–4× MAC density & effective BW per bit | accuracy risk; needs quantization ([Floating_Point](../00_Fundamentals/04_Floating_Point.md)) |
| Scale-out torus / NVLink (§6) | model parallelism beyond one die | collectives dominate; comm-bound at low intensity |

**The ledger, in real silicon: TPU vs GPU Tensor Core.** The two dominant training engines sit at opposite ends of the flexibility axis, and every row above predicts *where*. A **TPU** commits fully: one large systolic array (a $128\times128$ MXU) fed by a software-managed scratchpad — maximal MAC density and resident-operand reuse, but the compiler must map every layer (§2–§4) and a mismatched shape starves it. A **GPU** keeps the *same MAC leaf* but packages it as many small **Tensor-Core** MMA tiles per SM, fed from the register file and a hardware-cached shared memory beneath the SIMT scalar core — it sacrifices peak GEMM efficiency to keep the front end's flexibility, so it also runs the irregular shapes, control flow, and non-GEMM kernels a TPU cannot ([GPU_Architecture](15_GPU_Architecture.md)). Neither is "better": they are two different points on the efficiency–flexibility ledger, chosen for two different deployment bets.

| Axis | TPU (systolic) | GPU (SIMT + Tensor Core) |
|---|---|---|
| MAC organization | one large $D\times D$ array | many small MMA tiles per SM |
| Operand delivery | hardwired PE→PE + scratchpad | register file + cached shared mem |
| Who owns the mapping | compiler, statically (§4) | compiler *and* hardware, dynamically |
| Sweet spot | large regular GEMM, peak eff/W | mixed/irregular shapes, flexibility |

Where it stops winning: an NPU is efficient **only** in the compute-bound, high-reuse, regular-shape regime. Memory-bound operators (low operational intensity, §5), small or irregular shapes (poor utilization, §2), and control-heavy code (no dataflow to exploit) each erode the advantage back toward the HBM or the scalar unit, and the fixed datapath cannot adapt the way an out-of-order core does. This is exactly why real accelerators are *heterogeneous* (array + vector + scalar) and why the mapping/quantization/parallelism software stack is not optional polish but the thing that decides whether the silicon's peak is reachable at all. The corollary for a hardware architect is the co-design rule of [Full_Chip_Modeling §4.4](02_Full_Chip_Modeling.md): **size the array, the scratchpad, the HBM bandwidth, and the ICI together against a target workload's shapes and intensities — an array that outruns its SRAM or its HBM is starved silicon.**

---

## 8. Worked problems

**1 — Utilization is a bet on batch size (§2).** A $128\times128$ MXU ($D=128$). *(a)* A training tile $M=N=512,\ K=4096$: the edge factors are $512/(4{\cdot}128)=1$ each (512 is a multiple of 128), and fill/drain is $K/(K{+}2D)=4096/4352=0.94$, so $U\approx 94\%$. *(b)* A **batch-1 decode** GEMV $M=1,\ N=4096,\ K=4096$: the $M$ edge factor collapses to $1/(\lceil 1/128\rceil{\cdot}128)=1/128$, giving $U\approx (1/128)(1)(0.94)\approx 0.7\%$. The *same array* runs at 94% on training and under 1% on decode — a ~130× utilization swing from matrix shape alone. This is why the array size is a workload bet, and why serving stacks batch decode requests until $M$ approaches $D$.

**2 — Dataflow is an energy lever, quantified (§3).** Take a weight reused $r=256$ times as a long activation stream flows past it, with the hierarchy $e_{\text{DRAM}}{:}e_{\text{RF}}\approx 200{:}1$. Re-fetching it from DRAM every use (temporal, no reuse) costs $r\cdot e_{\text{DRAM}}=256\times200=51{,}200$ (in RF-access units). Making it **weight-stationary** costs $e_{\text{DRAM}}+(r{-}1)e_{\text{RF}}=200+255=455$ — a **$112\times$ energy reduction** on that operand, bought purely by *deciding which tensor stays resident*, with identical MAC count. This is why a dataflow is chosen against an energy model, not a cycle model.

**3 — Roofline predicts the regime before you build (§5).** An accelerator with $\pi=200$ TOPS (INT8) and $\beta=1$ TB/s HBM has ridge $I^{*}=\pi/\beta=200$ ops/byte. *(a)* A square GEMM $N=2048$ with 1-byte operands and good on-chip reuse moves $\approx 3N^2$ bytes off-chip for $2N^3$ ops, so $I=2N/3\approx 1365$ ops/byte $\gg I^{*}$ → **compute-bound**, the array is fed. *(b)* A batch-1 decode through a $d{\times}d$ weight layer reads $\approx d^2$ weight bytes to do $2d^2$ ops → $I\approx 2$ ops/byte $\ll I^{*}$ → **deeply memory-bound**, the MACs starve no matter how many you built. Same silicon, opposite regimes — and note it is the *same root cause* as problem 1: batch-1 has no reuse to capture, so both the array (utilization) and the HBM (roofline) report it. The roofline delivers that verdict from two numbers, before any RTL exists.

---

## Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Energy hierarchy (norm. to a MAC) | RF 1× · PE-PE 2× · SRAM buf 6× · DRAM 200× | why dataflow is an *energy* lever (Eyeriss) |
| DRAM vs on-chip access | ~100–200× (1–2 nJ vs ~10 pJ) | the entire data-reuse argument (Horowitz) |
| Instruction envelope vs MAC | arithmetic <~10% of the driving instruction's energy | why deleting per-MAC control wins (Horowitz) |
| GEMM work vs operands | $MNK$ MACs, $MK{+}KN{+}MN$ operands | reuse is huge and *static* → capturable on-chip |
| Systolic tile cost | $\approx K + 2D$ cycles ($D\times D$ array) | fill/drain → large tiles or wasted PEs |
| Utilization | edge-quant × $K/(K{+}2D)$ | array efficient only on large matrices |
| Batch-1 decode utilization | $\approx 1/D$ (~0.8% at $D{=}128$) | why decode underuses the array → must batch |
| TPUv1 MXU | $256{\times}256 = 65{,}536$ INT8 MACs @700 MHz = 92 TOPS | canonical systolic array; MAC density |
| Modern TPU MXU | $128{\times}128$ (v6e $256{\times}256$), 2–4 per TensorCore | smaller array = better real-shape utilization |
| On-chip scratchpad | TPUv1 24 MiB Unified Buffer (SW-managed) | explicit memory, not a cache |
| Roofline | $\min(\pi,\ \beta I)$, ridge $\pi/\beta$ | compute- vs memory-bound *before* you build |
| TPUv1 roofline ridge | ~1350 MAC/byte (92 TOPS ÷ 34 GB/s) | most operators sit left of it → memory-bound (Jouppi) |
| NVLink 5 / TPU pod | 1.8 TB/s per GPU / 9,216-chip torus | scale-out is interconnect-bound |
| Row-stationary edge | 1.4×–2.5× vs WS/OS (AlexNet) | dataflow is an energy, not a cycle, lever |

---

## Cross-references

- **Down the stack:** [Adders_and_Multipliers](../00_Fundamentals/03_Adders_and_Multipliers.md) (the MAC leaf), [Memory](09_Memory.md) (the SRAM the scratchpad is built from), [Cache_Microarchitecture](07_Cache_Microarchitecture.md) (the reactive machinery the scratchpad deliberately deletes), [Network_on_Chip](13_Network_on_Chip.md) (operand distribution on-die and torus routing), [Block_Activity_and_Power](../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (the activity × per-event-energy model behind §3), [Floating_Point](../00_Fundamentals/04_Floating_Point.md) (INT8/FP8/FP4 formats).
- **Up the stack:** [Performance_Modeling_and_DSE §10–11](01_Performance_Modeling_and_DSE.md) (the systolic cycle/utilization model, dataflow table, and NeuSim worked example), [Full_Chip_Modeling §4](02_Full_Chip_Modeling.md) (PE→array→chip→pod power/thermal, double-buffering, collectives), [Accelerator_and_NPU_Simulators](06_Simulators/05_Accelerator_and_NPU_Simulators.md) (how a layer + mapping becomes latency/energy — Timeloop/Accelergy, SCALE-Sim, ONNXim, NeuSim), [OoO_Execution §3](05_OoO_Execution.md) (the "what must the structure hold, and why" lens this page applies to the PE).
- **Adjacent (the other throughput accelerator):** [GPU_Architecture](15_GPU_Architecture.md) — SIMT and the Tensor-Core MMA leaf; the flexibility end of the efficiency–flexibility ledger whose specialized end this page's systolic array anchors (§7).
- **Companion AI-infra notebook (silicon-to-serving):** [Systolic_Arrays_and_Dataflow](../../silicon-to-serving/L2_Digital_Design_for_AI/03_Systolic_Arrays_and_Dataflow.md) (MXU internals, reuse arithmetic, input-stationary, Tenstorrent/Cerebras/NVIDIA/Ascend variants) and [Google_TPU](../../silicon-to-serving/L3_Microarchitecture/06_Google_TPU.md) (TPU generations, MXU, ICI 3D torus + OCS, XLA/JAX) — the vendor and MXU-internal depth this page assumes and does not duplicate.

## References

- H. T. Kung, C. E. Leiserson, "Systolic Arrays (for VLSI)," 1978; H. T. Kung, "Why Systolic Architectures?," *IEEE Computer* 1982 — [ieeexplore.ieee.org/document/1653825](https://ieeexplore.ieee.org/document/1653825/).
- N. P. Jouppi et al., "In-Datacenter Performance Analysis of a Tensor Processing Unit," ISCA 2017 — [arXiv:1704.04760](https://arxiv.org/abs/1704.04760).
- V. Sze, Y.-H. Chen, T.-J. Yang, J. S. Emer, "Efficient Processing of Deep Neural Networks: A Tutorial and Survey," *Proc. IEEE* 2017 — [arXiv:1703.09039](https://arxiv.org/abs/1703.09039).
- Y.-H. Chen, J. Emer, V. Sze, "Eyeriss: A Spatial Architecture for Energy-Efficient Dataflow for CNNs (row-stationary; Table IV energy hierarchy)," ISCA 2016 — [people.csail.mit.edu/emer/.../eyeriss_architecture.pdf](https://people.csail.mit.edu/emer/media/papers/2016.06.isca.eyeriss_architecture.pdf).
- M. Horowitz, "Computing's Energy Problem (and what we can do about it)," ISSCC 2014 — [gwern.net/doc/cs/hardware/2014-horowitz-2.pdf](https://gwern.net/doc/cs/hardware/2014-horowitz-2.pdf).
- S. Williams, A. Waterman, D. Patterson, "Roofline: An Insightful Visual Performance Model for Multicore Architectures," *CACM* 2009 — [people.eecs.berkeley.edu/~kubitron/cs252/handouts/papers/RooflineVyNoYellow.pdf](https://people.eecs.berkeley.edu/~kubitron/cs252/handouts/papers/RooflineVyNoYellow.pdf).
- "How to Think About TPUs" (JAX scaling book; modern MXU dimensions and systolic dataflow) — [jax-ml.github.io/scaling-book/tpus](https://jax-ml.github.io/scaling-book/tpus/).
- NVIDIA, "Fifth-Generation NVLink" (Blackwell, 1.8 TB/s per GPU; NVL72) — [nvidia.com NVLink](https://www.nvidia.com/en-us/data-center/nvlink/).
