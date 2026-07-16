# Performance Modeling and Design-Space Exploration

> **Stage:** 01 · Architecture & PPA (Performance, Power, Area) — the *performance* half of PPA, done **before any RTL (register-transfer level) exists**.
> **Prerequisites:** [Chip_Design_Flow_Overview](../Chip_Design_Flow_Overview.md), [CPU_Architecture](03_CPU_Architecture.md), [Memory](09_Memory.md).
> **Hands off to:** the µarch spec that [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) implements. **Owns the concepts; defers the machinery:** simulator internals live in [Simulation_Methodology](06_Simulators/01_Simulation_Methodology.md); full-chip composition, contention, and power/thermal coupling in [Full_Chip_Modeling](02_Full_Chip_Modeling.md); per-tool detail in the [Simulators folder](06_Simulators/00_Index.md).

---

## 0. Why this page exists

Architecture is a **one-way door**. You choose a microarchitecture — issue width, window depth, cache sizes, how many cores, what accelerator — and then RTL spends months and silicon spends a year *faithfully implementing that choice*. Every downstream stage optimizes the thing you committed to; none of them second-guesses whether it was the right thing. So the single most expensive mistake in chip design is made here, at the top, before a line of RTL exists — and it is made about a machine **that cannot yet be measured because it does not yet exist.**

If you cannot measure it, you **model** it. A model is an abstraction that deliberately throws away detail to answer a question faster than building the real thing would. That immediately sets up the organizing tension of this entire page — a **trilemma** between three things you cannot have at once:

- **Accuracy** — how close the model's number is to what the real silicon would do.
- **Speed** — how many design points you can evaluate per day.
- **Effort/coverage** — how much of the real machine (and how many workloads) the model captures.

You cannot maximize all three. A cycle-accurate model is accurate but slow; a spreadsheet is instant but coarse. The whole craft of pre-RTL modeling is choosing, for each *decision*, the point on that surface that is *just accurate enough to be right and no slower than necessary*. Stated as a rule that governs everything below:

> **Use the fastest, cheapest model whose resolution still exceeds the gap between the choices you are deciding between.**

If two cache sizes differ by 15% in performance and your model's error bar is ±30%, the model cannot see the difference — you must climb to a finer rung. If they differ by 2× , a spreadsheet settles it in a minute and cycle-accurate simulation is wasted effort. This page builds the tools that let you make that call: the **fidelity ladder** (§1), the **analytical kernels** that decide most questions on paper (§2, and their throughput-machine variants §9–§10), how and when to **climb to simulation** (§3–§4), how to search the exploding space with **design-space exploration** (§5), and how the **PPA axes actually trade** (§6). The recurring discipline is that performance is not one opaque number — it *decomposes*, and the decomposition tells you where to spend.

---

## 1. The modeling-fidelity ladder

Every model sits at a point on the **accuracy ↔ speed** curve, and the models form a ladder spanning roughly *six orders of magnitude* in simulation cost. You start at the top (cheapest) and climb only as far down as a decision forces you — because each rung costs ~10–100× more per evaluated point than the one above it.

| Rung | What it can resolve | Speed (host/target) | Error on cycles | Typical instance |
|---|---|---|---|---|
| **Analytical / mechanistic** | first-order: iron law, CPI stack, Amdahl, roofline; *which term dominates* | instant (closed form) | ±30–50% (ranking) | spreadsheet, interval model |
| **Trace-driven** | cache / branch / bandwidth behavior on a fixed address+branch stream | very fast | ±15–25% | cache sim on a trace |
| **Event-driven cycle-approximate** | pipeline + contention + reordering, abstracted | 10³–10⁴× slowdown | ±10–15% | gem5-O3, Sniper |
| **Cycle-accurate** | full timing validated *to a specific target* core | 10⁴–10⁵× slowdown | <5% vs that core | validated O3 config |
| **RTL / emulation** | the actual gates — it *is* the design | 10⁶–10⁷× (RTL); 10–10³× (FPGA) | exact | Verilator/VCS; [emulation](../03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md) |

Two properties make this a *ladder* rather than a menu:

- **"Cycle-accurate" is a claim about a target, not a virtue.** A model validated against a Cortex-A76 is not cycle-accurate for a Neoverse-V2. Accuracy is always *relative to a reference you calibrated against* — an uncalibrated "cycle-accurate" model can be 30% off (§8).
- **You live on the middle rungs because of the speed budget.** A design-space sweep of thousands of configurations cannot afford RTL; you push the physics *down* into pre-characterized per-event costs (cache access energies, DRAM timings) and keep the *composition* fast and cheap. This is why architects spend most of their time between "analytical" and "event-driven."

This page owns the **top rung** — the analytical kernels that decide the most and that every lower rung is a more-faithful realization of. *How* the lower rungs actually work — the discrete-event engine, trace- vs execution-driven feeding, region-of-interest sampling, warm-up, validation and error budgets — is the subject of [Simulation_Methodology](06_Simulators/01_Simulation_Methodology.md), and the ladder's use in composing a whole chip is [Full_Chip_Modeling §2](02_Full_Chip_Modeling.md). We cross-link that machinery rather than duplicate it.

---

## 2. Analytical models — the back-of-envelope that decides the most

The counter-intuitive truth of this stage: a spreadsheet, used well, settles more architecture questions than a simulator does. Analytical models are ±30–50% in *absolute* terms but far tighter in *ranking* — and ranking is what a design decision needs. Their real power is not the number but the *decomposition*: they tell you **which term dominates**, and therefore where a transistor buys performance and where it is wasted.

### 2.1 The iron law and the CPI stack

Start from the identity that underlies all processor performance — the **iron law**:

$$
T_{\text{program}} \;=\; \underbrace{IC}_{\text{ISA + compiler}} \;\times\; \underbrace{\text{CPI}}_{\text{microarchitecture}} \;\times\; \underbrace{t_{\text{cyc}}}_{\text{µarch + circuit + process}}
$$

where $IC$ = dynamic instruction count, $\text{CPI}$ = average cycles per instruction, $t_{\text{cyc}} = 1/f$ = clock period. Its value is *separation of concerns*: three factors owned by three different actors, and the architect owns the middle one outright and shares the third. It also warns you that the three are coupled — a deeper pipeline cuts $t_{\text{cyc}}$ but raises CPI (more mispredict exposure), so **you optimize the product, never one factor** (§2.4, §6).

The architect's factor, CPI, then decomposes **additively** into a base rate plus one stall term per hazard source:

$$\text{CPI} = \text{CPI}_{\text{base}} + \underbrace{m_{\text{L1}}\,p_{\text{L1}} + m_{\text{L2}}\,p_{\text{L2}} + m_{\text{mem}}\,p_{\text{mem}}}_{\text{memory}} + \underbrace{b\,(1-a)\,p_{\text{mispred}}}_{\text{branch}} + \dots$$

where $m_x$ = misses per instruction at level $x$, $p_x$ = penalty cycles at level $x$, $b$ = branch fraction, $a$ = predictor accuracy. This is the **CPI stack**, and its two gifts are the reason it is the workhorse of the whole page:

1. **Additive → attributable.** Each term is a bar in a stacked chart. The tallest bar *is* the bottleneck, named and quantified. If memory-CPI swamps branch-CPI, a fancier predictor is wasted silicon no matter how elegant — spend the area on the cache or the prefetcher. Every activity counter that feeds a power model comes from this same decomposition, which is why [Full_Chip_Modeling §1.5](02_Full_Chip_Modeling.md) drives McPAT straight off the CPI-stack counters.
2. **Actionable → marginal reasoning.** Because the terms add, the *marginal* value of any lever is the reduction in its own term. You spend where $\partial \text{CPI}/\partial(\text{area})$ is largest — never uniformly.

**The theoretical catch that separates a novice from an architect:** the additive stack assumes each stall is *fully exposed*, which is true for an in-order or blocking machine but **wrong for out-of-order execution**. An OoO core overlaps independent misses (memory-level parallelism, MLP) and hides stalls under other work, so the *exposed* penalty is far below the *raw* latency:

$$
\text{CPI}_{\text{stall}} \;=\; \sum_{e} N_e \, p_e^{\text{exposed}}, \qquad p_e^{\text{exposed}} \;=\; \frac{p_e^{\text{raw}}}{\text{MLP}_e}\;\ (\le p_e^{\text{raw}})
$$

where $N_e$ = events of type $e$ per instruction, $p_e^{\text{raw}}$ = full latency, $\text{MLP}_e$ = independent events of that type overlapped in the window. The additive stack with raw latencies is therefore an **upper bound** on stall CPI; multiplying MPKI by raw DRAM latency systematically *over*-counts, sometimes absurdly (the CPI goes negative — §8). This single subtlety is why the honest analytical model carries an overlap factor, and why the cycle-approximate rung exists: to *measure* the MLP the closed form can only estimate. (The mechanistic/interval model that makes this rigorous is [Analytical_Models](06_Simulators/07_Analytical_Models.md).)

### 2.2 Amdahl and the parallel ceiling

$$\text{Speedup} = \frac{1}{(1-p) + p/N}$$

where $p$ = parallelizable fraction of the work, $N$ = number of parallel units. The serial fraction $1-p$ caps everything: at $p=0.9$, infinite cores give only $10\times$. Amdahl is *why* accelerators co-design the algorithm, not just the hardware — you attack the serial residue, because no amount of $N$ will. It is the pure-parallelism ceiling; the real multicore curve is lower still once shared-resource **contention** is added as a growing term $C(N)$ (the Universal Scalability Law), which is developed where contention lives — [Full_Chip_Modeling §2.2](02_Full_Chip_Modeling.md) and [Analytical_Models](06_Simulators/07_Analytical_Models.md).

### 2.3 Roofline — compute-bound or memory-bound, before you size anything

A kernel of **arithmetic intensity** $I$ (useful FLOPs per byte moved) on a machine of peak compute $\pi$ (FLOP/s) and peak bandwidth $\beta$ (byte/s) can attain at most

$$P_{\text{attainable}} \;=\; \min\!\big(\pi,\ \beta \cdot I\big)$$

Two roofs: a flat **compute roof** $\pi$ and a slanted **memory roof** $\beta I$. They cross at the **ridge point**

$$I^\star = \pi / \beta \quad\text{(FLOP/byte)},$$

which partitions all kernels: $I < I^\star$ ⇒ **memory-bound** (you are on the slanted roof; more FLOPS buys nothing — raise $I$ by fusion/blocking, or raise $\beta$); $I > I^\star$ ⇒ **compute-bound** (raise $\pi$ or utilization). Roofline is a *ceiling*, not a prediction — real performance sits below it whenever latency is not hidden (insufficient MLP/occupancy, §2.1, §9.1) — but it tells you *which knob is even live* before you spend a transistor. It is the AI-era PPA lens, applied **per kernel** for throughput machines (§9.3, §10.3).

### 2.4 Worked mini-example — issue width vs frequency (DSE in one decision)

A 4-wide core at 3 GHz vs. a 2-wide core at 4 GHz (the shorter pipeline that bought the clock also raised mispredict exposure, hurting IPC). Suppose the 4-wide sustains IPC 1.8, the 2-wide IPC 1.2:

- 4-wide: $1.8 \times 3 = 5.4$ BIPS (billion instructions/s). 2-wide: $1.2 \times 4 = 4.8$ BIPS.

Raw throughput favors the 4-wide — but this is a trap, because the *decision variable is not BIPS*. Power scales as $\sim W\!\cdot\! f\!\cdot\! V^2$ and the 4-wide needs more area (superscalar structures grow super-linearly, [OoO_Execution §4.3](05_OoO_Execution.md)). The architect compares **BIPS/W and BIPS/mm²**, not BIPS — because power and area are *budgeted*, and the winner is the point on the Pareto frontier (§5) that meets the binding constraint. This one example is the whole method in miniature: the iron law sets the objective, and PPA efficiency (§6) — not raw performance — decides.

---

## 3. Climbing to cycle simulation — when the spreadsheet can't decide

The analytical stack fails precisely where behaviors *interact*: contention for a shared port or channel, out-of-order reordering that changes what overlaps, a prefetcher racing a stride, wrong-path instructions polluting a cache. These are **emergent** — they are not the sum of independent terms, so no closed form captures them. When two candidate designs differ only in such an interaction, you climb to the **event-driven cycle-approximate / cycle-accurate** rung and *simulate*.

Conceptually a timing simulator makes the CPI stack **executable**: instead of estimating each stall term, it maintains the actual microarchitectural structures (reorder buffer, issue queue, load-store queue, MSHRs, cache hierarchy, branch predictor) and lets throughput fall out of *whichever structure saturates first*. That is the payoff — you learn not just that IPC is 1.8 but *why* (ROB-full? LSQ-full? a long dependence chain? issue-port pressure?), which the additive stack cannot tell you because it assumed the terms were independent.

The two facts an architect must carry from this rung (both derived in depth in [Simulation_Methodology](06_Simulators/01_Simulation_Methodology.md), not repeated here):

- **A simulated number is only as trustworthy as its provenance.** Which rung produced it, and was it *validated and calibrated* against a reference? An uncalibrated architectural model can be 2× off; a calibrated one ~20–30% (§8). Always ask the provenance before believing the digit.
- **You never run the whole workload.** A trillion-instruction SPEC run at 10⁴× slowdown is months of wall-clock, so the reported number is the product of a **sampling** methodology (SimPoint phase clustering, SMARTS statistical sampling) plus **warm-up** — and most simulation *mistakes* live in that methodology, not in the model. This is exactly why cheap analytical pruning (§5) precedes expensive simulation.

The tool that instantiates this rung for CPUs is **gem5** (configurable in-order/O3 CPU models, classic/Ruby memory); its configuration and validation are covered in [gem5](06_Simulators/02_gem5.md). We deliberately do **not** reproduce its config knobs here — the modeling *decision* (when to climb, and what the rung buys) is the concept; the tool syntax is a lookup.

---

## 4. System-level modeling and virtual platforms — a different question

Not every model is about a core's CPI. For a full **SoC** (system-on-chip) — CPUs, DMA (direct memory access) engines, accelerators, a NoC (network-on-chip), memory controllers — there is a second modeling *purpose* that has nothing to do with cycle accuracy: **enabling software and interconnect analysis before RTL exists.** This is transaction-level modeling (**SystemC / TLM-2.0**), and it trades away pipeline detail for whole-system reach at two fidelities:

- **Loosely-timed (LT)** — blocking transactions, temporal decoupling; fast enough to **boot the OS and run firmware** on a *virtual platform* months before RTL. The deliverable is time-to-software, not an accurate cycle count: driver and firmware teams start against the virtual platform instead of waiting a year for silicon.
- **Approximately-timed (AT)** — non-blocking, models arbitration/contention/latency phases; used for early **bandwidth/latency** estimates of the bus and memory system ([AHB_AXI_APB](11_AHB_AXI_APB.md), [DDR_Controller](10_DDR_Controller.md)).

The point for this page is that "modeling" is not one axis. §2–§3 climb the *accuracy* axis for a core; virtual platforms climb a *coverage/time-to-software* axis for a system. The systematic composition of leaf models into a full-chip power+performance model — and the contention, DVFS, and thermal coupling that only appear at that scale — is the subject of [Full_Chip_Modeling](02_Full_Chip_Modeling.md).

---

## 5. Design-space exploration — optimization over the PPA surface

Every knob above defines an axis; together they define a **design space**, and DSE is the search over it for the PPA-optimal points. The reason DSE needs a *methodology* rather than brute force is that the space is a Cartesian product:

$$
|\mathcal{S}| \;=\; \prod_{k=1}^{K} n_k
$$

where $K$ = number of parameters (width, ROB depth, L2 size, associativity, frequency, $V_t$ mix, #cores, NoC topology, …) and $n_k$ = options for knob $k$. This is **exponential in the number of knobs**: a modest 8 knobs at 4 settings each is $4^8 \approx 65{,}000$ points, and at cycle-accurate fidelity (hours per point) that is years of compute. The governing inequality of DSE is a budget constraint,

$$
\underbrace{|\text{points evaluated}|}_{\text{coverage}} \;\times\; \underbrace{\text{cost per evaluation}}_{\text{fidelity}} \;\le\; \text{compute budget},
$$

and it forces the entire recipe — you buy coverage by making evaluations cheap:

1. **Prune analytically.** Kill obviously-dominated points at ~zero cost with the §2 kernels (a config that is memory-bound *and* has a small cache cannot win). This removes most of $\mathcal{S}$ before any simulation.
2. **Cycle-simulate the survivors.** Spend the expensive rung only where the analytical model could not resolve the ranking.
3. **Guide the sampling.** When even the survivor set is too large, a **surrogate** (Bayesian optimization / ML model of the PPA surface) is fit to a handful of samples and used to pick the next most-informative point — sample-efficient search over an expensive black box.

The output is not a single answer but a **Pareto frontier**: the set of points where no other point is better in *all* of performance, power, and area simultaneously (no point *dominates* them). The architect then picks by the **binding constraint** — "minimum power that still meets 5.4 BIPS," or "max perf under 2 W and 10 mm²."

```mermaid
%%{init: {"flowchart": {"defaultRenderer": "elk", "nodeSpacing": 60, "rankSpacing": 60, "htmlLabels": false}}}%%
flowchart TD
    P["Parameter space<br/>(width, ROB, L2, f, Vt mix...)<br/>|S| = product of knobs — exponential"]:::a --> PR["Prune analytically<br/>(§2 kernels kill dominated points)"]:::b
    PR --> S["Cycle-simulate survivors<br/>(guided by surrogate / BayesOpt)"]:::c
    S --> M["PPA metrics<br/>perf, power, area"]:::d
    M --> F["Pareto frontier<br/>(no point dominates)"]:::e
    F --> CH["Pick by binding constraint<br/>(meet perf @ min power)"]:::f
    classDef a fill:#fde68a,stroke:#b45309,color:#000
    classDef b fill:#fed7aa,stroke:#c2410c,color:#000
    classDef c fill:#bbf7d0,stroke:#15803d,color:#000
    classDef d fill:#bae6fd,stroke:#0369a1,color:#000
    classDef e fill:#c7d2fe,stroke:#4338ca,color:#000
    classDef f fill:#fbcfe8,stroke:#9d174d,color:#000
```

At industrial scale this exact recipe is realized with a distributed job scheduler sweeping millions of configurations — the concrete instance is the NeuSim DSE of §11.

---

## 6. The PPA trade-off — the fundamental tension

Performance, power, and area are not three independent dials; they are **coupled through the same physical budget** — transistors, watts, and mm² are all finite and all spent from one pool. You cannot move one without paying in the others, so design is always motion *along a surface*, never toward a corner. The levers and their signs:

| Lever | Performance | Power | Area | Why / where it saturates |
|---|---|---|---|---|
| ↑ Frequency | + | ++ ($V^2$, then thermal) | ~0 | hits the [power wall](../02_Power_and_Low_Power/01_Power_Fundamentals.md) fast; $P\sim f^3$ over the DVFS range |
| ↑ Issue width / OoO depth | + (diminishing) | + | ++ | wakeup/RF port cost grows $\sim W^2$ ([OoO §4.3](05_OoO_Execution.md)) |
| ↑ Cache size | + (until working set fits) | + (leakage) | ++ | the classic area sink; step-function returns |
| Pipeline deeper | + $f$, − IPC (mispredict) | + | + | net perf **non-monotonic** (§2.4) |
| Specialize (accelerator) | +++ on target | −− energy/op | + | the AI-chip bet; useless off-target (Amdahl) |
| Parallelize (more cores) | + (Amdahl-capped) | + | ++ | only if the workload scales; contention $C(N)$ |

The deep point: because power and area are *budgeted*, the true objective is almost never raw performance but **performance per watt** and **performance per mm²** — a memory-bound design that doubles FLOPS at 2× power moved *backward*. And the architect's allocation rule is the CPI-stack/roofline rule from §2: **spend each mm² and each mW where the decomposition says the bottleneck is, never uniformly.** DSE (§5) formalizes this as picking a Pareto point; §2–§4 are the models that place a candidate on the surface.

---

## 7. Numbers to memorize

| Quantity | Value | Why it matters (section) |
|---|---|---|
| Analytical model error | ±30–50% | good for *ranking*, not signoff (§2) |
| Trace-driven error | ±15–25% | cache/branch studies (§1) |
| Cycle-approximate error | ±10–15% | the working DSE rung (§1, §3) |
| Cycle-accurate error (validated) | ~5% | the architecture commit point (§1) |
| Uncalibrated architectural model | up to ~2× off | why calibration/validation is mandatory (§3, §8) |
| gem5 O3 speed | ~0.1–1 MIPS | why sampling exists (§3) |
| SimPoint coverage | ~1% of instrs simulated | bounded-error extrapolation (§3) |
| Amdahl at $p=0.9$ | 10× ceiling | serial fraction dominates (§2.2) |
| Superscalar IPC reality | ~1–2 sustained (general code) | width has diminishing returns (§2.4, §6) |
| Roofline ridge point | $I^\star=\pi/\beta$ | compute- vs memory-bound split (§2.3) |
| TLM-LT use | boots OS pre-RTL | early SW on a virtual platform (§4) |
| Systolic fill/drain | $K/(K{+}2D)$ amortization | small reductions waste the array (§10.1) |
| DSE space size | $\prod_k n_k$ (exponential in knobs) | why you prune + sample (§5) |

**Memory hierarchy latencies** (the $p_x$ behind the CPI stack): register ~1 cycle · L1 ~4 · L2 ~10–15 · L3 ~30–50 · **DRAM ~100–300 cycles** — the term that dominates memory-CPI and drives the roofline for most modern workloads.

---

## 8. Worked problem — the MLP trap

**Q.** Architecture proposes doubling L2 from 1 MB to 2 MB. Trace-driven sim shows L2 MPKI drops 12 → 7. L2 penalty is 12 cyc, mem penalty 200 cyc; the extra MB adds 1.5 mm² and 60 mW leakage. The core runs at IPC 1.5, 3 GHz. Worth it?

*Naive solution.* The 5 fewer misses/1000-instr now hit L2 instead of memory, saving $(200-12)=188$ cyc each → $\Delta\text{CPI} = \tfrac{5}{1000}\times 188 = 0.94$ cyc/instr. But base CPI is only $1/1.5 = 0.67$, so "new CPI $= 0.67 - 0.94 < 0$" — **impossible.**

*The lesson (this is §2.1's caveat in the flesh).* The contradiction proves the additive stack with *raw* latency over-counts: on an OoO core those misses were **partly overlapped** (MLP), so the *exposed* penalty was far below 188 cycles — you cannot recover cycles the machine never actually lost. The correct move is to plug the MPKI change into the CPI stack **with a realistic overlap/MLP factor** (or, better, re-run the cycle model, which measures the overlap instead of guessing it), then judge the gain on **BIPS/W and BIPS/mm²** against the Pareto frontier (§5–§6). If the re-run shows >~3–5% performance for +1.5 mm²/+60 mW, it likely earns its place. The takeaway an architect keeps: **never multiply MPKI by raw latency on an out-of-order machine.**

---

## 9. GPU performance modeling — throughput, occupancy, per-kernel roofline

Everything above is CPU-centric: the metric is **latency of one instruction stream** (CPI, IPC). A GPU inverts the philosophy — it is a **throughput** machine that *hides* latency with thousands of threads instead of *avoiding* it — so the model changes shape. The unit of analysis is the **kernel**, and the bottleneck is almost always *occupancy* or *memory bandwidth*, not per-instruction stalls. (The hardware that these models describe is [GPU_Architecture](15_GPU_Architecture.md); how a GPU is *simulated* is [GPU_Simulators](06_Simulators/04_GPU_Simulators.md).)

### 9.1 Occupancy and latency hiding

Threads run in **warps** (32 lanes, SIMT — single-instruction, multiple-thread); a streaming multiprocessor (SM) round-robins ready warps to fill stall bubbles. **Occupancy** = resident warps / max warps per SM, capped by whichever per-SM resource runs out first:

$$\text{occupancy} = \frac{\min\!\left(\underbrace{\tfrac{\text{regs}_{\max}}{32\cdot\text{regs/thread}}}_{\text{reg-limited warps}},\ \underbrace{\left\lfloor\tfrac{\text{smem}_{\max}}{\text{smem/block}}\right\rfloor\cdot\tfrac{\text{warps}}{\text{block}}}_{\text{smem-limited warps}},\ W_{\max}\right)}{W_{\max}}$$

(each term is *warps per SM* — registers are budgeted per 32-thread warp, shared memory per resident block — and hardware also caps resident blocks/SM.) The load-bearing theory is **Little's law**: to hide a memory latency $L$ at issue throughput $\lambda$ you need

$$\text{warps} \;\gtrsim\; L \times \lambda \quad\text{(operations in flight = throughput × latency)}.$$

Too few warps → the SM stalls with no ready work even though peak FLOPS is untouched (latency-bound, not compute-bound). The lever is register/shared-memory pressure, not clock. Crucially, occupancy is a *ceiling on latency-hiding, not throughput itself*: past the point where warps cover the latency, more occupancy buys nothing — you are then set by which side of the roofline knee (§2.3, §9.3) you are on.

### 9.2 Memory coalescing

DRAM/L2 sees **memory transactions**, not threads. When a warp's 32 threads touch one aligned cache line, the access **coalesces** into a single transaction; strided/scattered access fans out into many, multiplying traffic and collapsing achieved bandwidth. Coalescing efficiency $=$ requested bytes / transferred bytes is a first-order multiplier on $\beta$ in the roofline — a strided kernel can sit 8–32× below peak BW with *no change in compute*, moving it left of the ridge point into memory-bound territory.

### 9.3 Per-kernel roofline

Apply roofline (§2.3) **per kernel**, with $I$ = FLOPs / bytes computed *after* coalescing and cache effects. A GEMM (general matrix multiply) kernel sits near the compute roof; an element-wise or LayerNorm kernel sits far left (memory-bound) — they need *opposite* optimizations, which is why you never roofline "the GPU," you roofline each kernel. The model tells you which knob is live (occupancy vs intensity vs bandwidth) before you touch a line of CUDA. This is the same knee that, at chip scale, makes adding SMs past $N^\star = B_{HBM}/\text{demand}$ pure waste ([Full_Chip_Modeling §3.3](02_Full_Chip_Modeling.md)).

### 9.4 Worked example — occupancy-bound vs memory-bound

An SM allows 64 warps (2048 threads), 64K registers, 96 KB shared memory. A kernel uses **64 registers/thread**, 256-thread blocks.

- Register-limited: $65536 / (64 \times 32) = 32$ warps (a warp = 32 threads → $64\times32=2048$ regs/warp; $65536/2048=32$). Occupancy $= 32/64 = 50\%$.
- Recompiling to 32 regs/thread → 64 warps → **100% occupancy**, doubling latency-hiding capacity — *but only helps if the kernel is latency-bound.* If it is memory-bandwidth-bound, higher occupancy buys nothing once BW saturates.
- Roofline check: a GEMM tile at $I=40$ FLOP/byte on a 2 TB/s, 60 TFLOP/s part has ridge $I^\star=\pi/\beta=30$. Since $40>30$ → **compute-bound** → fix occupancy/instruction mix. An element-wise kernel at $I=0.25$ is hopelessly memory-bound → **fuse it to raise $I$**; occupancy is irrelevant. The model chooses the knob before any code changes.

---

## 10. NPU / accelerator performance modeling — systolic arrays, dataflow, tiling

An NPU/TPU (neural / tensor processing unit) discards the thread-scheduler abstraction entirely: the workhorse is a **systolic array (SA)** of $D\times D$ MAC (multiply-accumulate) PEs (processing elements) streaming a tiled GEMM, plus a SIMD **vector unit (VU)** for non-GEMM ops and an SRAM/HBM hierarchy. Performance modeling becomes a **loop-nest + utilization** problem, not an IPC problem — the entire computation is known statically from the layer shape, so it is an *offline analysis*, not an execution. (The hardware is [NPU_Accelerators](16_NPU_Accelerators.md); the mapping-to-PPA tools are [Accelerator_and_NPU_Simulators](06_Simulators/05_Accelerator_and_NPU_Simulators.md).)

### 10.1 The systolic-array cycle and utilization model

A $D\times D$ array computing $C_{M\times N}=A_{M\times K}B_{K\times N}$, tiled into $D\times D$ output tiles with $K$ streamed, must **fill** and **drain** the pipeline around the $K$ useful cycles per tile:

$$\text{cycles}_{\text{tile}} \approx K + 2D \quad(\text{fill + drain} \approx 2D), \qquad \text{cycles}_{\text{total}} \approx \left\lceil\frac{M}{D}\right\rceil\left\lceil\frac{N}{D}\right\rceil\,(K + 2D)$$

The $2D$ is a first-order stand-in (an exact wavefront gives $(D{-}1)+(D{-}1)=2D-2$; some formulations bill $\approx D$ when fill and drain overlap across back-to-back tiles) — use it to *rank* mappings and expose the penalty, not for cycle-exact signoff. **Utilization** = useful MACs / (capacity × cycles) factors into exactly two losses:

$$U = \frac{M\,N\,K}{D^2 \times \text{cycles}_{\text{total}}} \;=\; \frac{1}{\underbrace{\left\lceil M/D\right\rceil D / M \cdot \left\lceil N/D\right\rceil D / N}_{\text{edge-tile quantization}}}\times\underbrace{\frac{K}{K+2D}}_{\text{fill/drain amortization}}$$

**Edge quantization** (small/odd $M,N$ don't fill the array — pad waste) and **fill/drain** (small $K$ can't amortize the $2D$ latency) both push toward large, array-aligned tiles. This is the single formula the hardware and simulator pages both build on.

### 10.2 Dataflow taxonomy — mostly an energy lever

*Which* tensor stays resident in the PEs sets reuse, SRAM traffic, and which dimension amortizes the pipeline:

| Dataflow | Stationary operand | Reuse maximized | Typical fit |
|---|---|---|---|
| **Weight-stationary (WS)** | weights held in PEs | weight reuse across the activation stream | TPU-style GEMM, many activations per weight |
| **Output-stationary (OS)** | partial sums accumulate in place | psum reuse (no read-modify-write to SRAM) | deep reduction — large $K$ amortized in-place |
| **Row-stationary (RS)** | a 1-D conv row mapped into a PE (Eyeriss) | balances weight/act/psum reuse | CNNs, low data-movement energy |

The dataflow changes *cycles* only modestly (via utilization) but changes *energy* substantially (via which memory level absorbs the accesses, and $e_{\text{DRAM}}\gg e_{\text{SRAM}}\gg e_{\text{RF}}$). Eyeriss's row-stationary was 1.4×–2.5× more energy-efficient than the alternatives on AlexNet — a scoped, workload-specific result, not a global optimum. This is *why* dataflow selection is driven by an analytical *energy* model, with cycles as the confirmation.

### 10.3 Tiling and the three bottleneck classes

The mapper picks tile sizes ($T_m,T_n,T_k$) so each tile's operands fit in SRAM, then orders the loop nest. The **roofline still rules**, per operator, with three architecture-specific bottleneck classes:

- **SA-bound** — the GEMM saturates the MAC array; you are at the compute roof. *Fix:* bigger array, more SAs, higher $U$.
- **VU-bound** — softmax / LayerNorm / activation on the vector unit is the critical path (common in attention). *Fix:* more VU lanes, fuse into the GEMM epilogue.
- **HBM-bound** — operand/weight traffic exceeds HBM bandwidth (low arithmetic intensity, KV-cache reads). *Fix:* bigger tiles for reuse, more HBM BW, quantization.

This SA/VU/HBM taxonomy is what an operator-level tool emits per operator (§11) and what the full-chip model composes ([Full_Chip_Modeling §4](02_Full_Chip_Modeling.md)).

### 10.4 Worked example — GEMM on a 128×128 array

Map $C=AB$ with $M=512, N=512, K=256$ onto $D=128$ (WS):

- Tiles: $\lceil 512/128\rceil^2 = 4\times4 = 16$ output tiles, each $128\times128$.
- Per-tile cycles $\approx K + 2D = 256 + 256 = 512$; total $\approx 16\times512 = 8192$ cycles.
- Useful MACs $= MNK = 6.71\times10^7$; capacity·cycles $= D^2\times8192 = 1.34\times10^8$.
- **Utilization** $U = 6.71\times10^7 / 1.34\times10^8 = \mathbf{50\%}$, and the cause is explicit: $M,N$ tile cleanly (no edge loss), so the entire 50% is **fill/drain** — $K/(K+2D)=256/512=0.5$. Doubling $K$ to 512 lifts $U$ to $512/768 = 67\%$; **tiling the $K$ loop deeper** (a longer reduction per array load) is the lever. At 1 GHz the kernel is $8192$ ns and **SA-bound** — but if a fused softmax afterward took $>8192$ cycles on the VU, the operator flips **VU-bound** and the array sizing is wasted.

---

## 11. Worked example — building an industrial operator-level DSE model (NeuSim)

The capstone question: **how do the analytical kernels of §9–§10 become a real industrial DSE tool?** [NeuSim](https://github.com/platformxlab/NeuSim) (UIUC PlatformX; backs the Neu10/MICRO'24, V10/ISCA'23, and ReGate/MICRO'25 papers) is an open-source, config-driven NPU simulator that runs the full **perf → power → carbon → SLO** (service-level-objective) loop at design-space scale. It is worth studying not for its command syntax — trimmed here on purpose — but because it makes concrete the *three ingredients every industrial DSE tool has*.

> **Scope boundary.** Composing these operator-level results *up* the hierarchy — core→chip→pod, with contention, compute/memory/comm overlap, DVFS/turbo, and thermal throttling co-modeled — is [Full_Chip_Modeling §4](02_Full_Chip_Modeling.md). NeuSim's place in the *simulator* taxonomy (operator-level vs cycle-accurate vs analytical-mapping) is [Accelerator_and_NPU_Simulators §7](06_Simulators/05_Accelerator_and_NPU_Simulators.md). This section owns the "how you build one" concept.

**Ingredient 1 — one perf model per block, reporting the bottleneck class as output.** NeuSim models each component with exactly the §10 abstraction, and for **each tensor operator** reports execution time, FLOPS, and memory traffic, and **classifies it SA-/VU-/HBM-bound** — the §10.3 taxonomy emitted automatically instead of hand-derived:

| Component | Modeled quantity | Maps to |
|---|---|---|
| **Systolic array (SA)** | GEMM cycles, FLOPS, utilization | §10.1 (SA-bound) |
| **Vector unit (VU)** | SIMD time for non-GEMM ops | §10.3 (VU-bound) |
| **On-chip SRAM** | capacity, reuse, DMA staging | tiling / §10.3 |
| **HBM** | bandwidth, memory traffic | §10.3 (HBM-bound) |
| **ICI (inter-chip interconnect)** | collective BW across a pod (2D/3D torus) | multi-chip parallelism |

**Ingredient 2 — an energy/carbon overlay on top of the perf activity.** The tool separates concerns into modular stages, each consuming the previous stage's output, so you can re-run power without re-running performance: per-operator **performance** → per-component **energy** (static + dynamic) → **carbon** (embodied + operational, from energy × carbon-intensity × duty cycle) → **SLO** filtering (which configs meet a latency target). This is the accelerator analogue of "gem5 → McPAT → power/thermal → does-it-meet-spec," but operator-level and carbon-aware — the same activity × per-event-energy pattern McPAT uses for CPUs ([Full_Chip_Modeling §1.5](02_Full_Chip_Modeling.md)).

**Ingredient 3 — a config-swept, distributed search.** The design space is three orthogonal config axes swept combinatorially — this is §5's DSE made real for NPUs:

- **hardware** — #SAs, #VUs, core frequency, HBM bandwidth, SRAM size;
- **model** — the network graph + parallelism strategy (tensor/pipeline/data/expert, TP/PP/DP/EP);
- **system** — datacenter PUE and carbon intensity (the carbon axis).

Sweeping *#chips × batch × NPU-version × parallelism* explodes into millions of jobs, so a distributed scheduler (Ray) fans them across machines — the practical answer to "the space is combinatorial, you can't brute-force cycle-accurate" (§5). The output is a Pareto set the SLO stage filters to the most cost-/carbon-efficient config meeting the target.

NeuSim also exposes power-side knobs the static roofline cannot see — **power gating** and **DVFS** per component — which capture leakage burned in the idle bubbles *between* operators (30–72% of NPU energy is static; ReGate, MICRO'25). Modeling those is what turns a perf model into a perf-*and*-power co-model; the power-side detail is [Block_Activity_and_Power](../02_Power_and_Low_Power/02_Block_Activity_and_Power.md).

**The lesson for an architect:** an industrial DSE tool is **(component perf models) + (an energy/carbon overlay) + (a config-swept, distributed search)** — nothing more exotic. The §2/§9/§10 analytical kernels become the per-component models, §5 DSE becomes the distributed sweep, and §6 PPA trade-offs become the SLO/carbon Pareto filter. Every piece on this page is a piece of that tool.

---

## Cross-references

- **Down the stack (what these models are built from):** [CPU_Architecture](03_CPU_Architecture.md) (the iron law and pipeline the CPI stack decomposes), [Memory](09_Memory.md) & [DDR_Controller](10_DDR_Controller.md) (the $p_{\text{mem}}$ latencies and bandwidth roofs), [OoO_Execution](05_OoO_Execution.md) (the window-sizing that sets MLP and the $\sim W^2$ issue cost behind §2.4/§6).
- **Sideways (the machinery this page defers to):** [Simulation_Methodology](06_Simulators/01_Simulation_Methodology.md) (how the lower rungs actually work — event engine, sampling, validation), [gem5](06_Simulators/02_gem5.md) / [GPU_Simulators](06_Simulators/04_GPU_Simulators.md) / [Accelerator_and_NPU_Simulators](06_Simulators/05_Accelerator_and_NPU_Simulators.md) (per-tool detail), [Analytical_Models](06_Simulators/07_Analytical_Models.md) (roofline/interval/Little's-law/USL formalized), [Full_Chip_Modeling](02_Full_Chip_Modeling.md) (composing leaves into a chip; contention, DVFS, thermal, tool map).
- **Up the stack (what consumes this page):** [RTL_Design_Methodology](../03_Frontend_RTL_and_Verification/01_RTL_Design_Methodology.md) (implements the committed µarch spec), [Gate_Level_Sim_and_Emulation](../03_Frontend_RTL_and_Verification/13_Gate_Level_Sim_and_Emulation.md) (validates the perf model against real workloads), [Power_Fundamentals](../02_Power_and_Low_Power/01_Power_Fundamentals.md) & [Block_Activity_and_Power](../02_Power_and_Low_Power/02_Block_Activity_and_Power.md) (the power half of PPA), [GPU_Architecture](15_GPU_Architecture.md) & [NPU_Accelerators](16_NPU_Accelerators.md) (the throughput/dataflow hardware §9–§10 model).

---

## References

1. Hennessy, J.L. and Patterson, D.A., *Computer Architecture: A Quantitative Approach*, 6th ed., Morgan Kaufmann, 2017. The iron law, CPI, Amdahl, and the limits of the window.
2. Williams, S., Waterman, A., Patterson, D., "Roofline: An Insightful Visual Performance Model for Multicore Architectures," *CACM*, 52(4), 2009. The roofline of §2.3/§9.3/§10.3.
3. Karkhanis, T. and Smith, J.E., "A First-Order Superscalar Processor Model," *ISCA*, 2004. The mechanistic/interval CPI model behind §2.1's overlap correction.
4. Sherwood, T. et al., "Automatically Characterizing Large Scale Program Behavior (SimPoint)," *ASPLOS*, 2002; Wunderlich, R. et al., "SMARTS: Accelerating Microarchitecture Simulation via Rigorous Statistical Sampling," *ISCA*, 2003. The sampling of §3.
5. Chen, Y.-H., Emer, J., Sze, V., "Eyeriss: A Spatial Architecture for Energy-Efficient Dataflow for CNNs (row-stationary)," *ISCA*, 2016. The §10.2 dataflow result.
6. Jouppi, N.P. et al., "In-Datacenter Performance Analysis of a Tensor Processing Unit," *ISCA*, 2017. The systolic GEMM machine of §10.
7. NeuSim / PlatformX (UIUC) — [github.com/platformxlab/NeuSim](https://github.com/platformxlab/NeuSim); ReGate, MICRO 2025 (NPU static-power fraction). The §11 industrial DSE reference.
