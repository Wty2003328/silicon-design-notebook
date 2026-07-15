# Analytical models — the closed-form duals of the simulators

> **Prerequisites:** [Simulation_Methodology](01_Simulation_Methodology.md) (why a closed-form is the top rung of the fidelity ladder), [Performance_Modeling_and_DSE](../01_Performance_Modeling_and_DSE.md) (the CPI stack §2.1, Amdahl §2.2, roofline §2.3/§9.3, occupancy/Little §9.1 — **this page extends those kernels, it does not repeat them**).
> **Hands off to:** [gem5](02_gem5.md) and the other per-tool pages — the executable models you escalate to when the closed form stops distinguishing your choices.

---

## 0. Why this page exists

Every simulator in this folder is an *executable* model; each has a **closed-form dual** that returns the answer on a whiteboard, to ~10–20%, in the time it takes to divide two numbers — and, more valuably, tells you **which knob is live before you spend a CPU-week simulating.** The [Performance_Modeling_and_DSE](../01_Performance_Modeling_and_DSE.md) page introduced the working kernels (CPI stack, Amdahl, roofline, occupancy). This page is their **rigorous back**: the ceilings and cache-aware form of roofline, the full mechanistic CPI equation behind the interval model, Amdahl's missing *contention* term, and the **queueing spine** (Little + M/M/1) that every contention model in a simulator is secretly a structural realization of. The organizing claim: **analytical models bound, decompose, and size; simulators only tell you the same thing more slowly and more accurately.** Knowing the dual is what lets you *read* a simulator's output instead of merely collecting it.

---

## 1. What the analytical layer is for — bound, decompose, size, escalate

Analytical models do three jobs a simulator also does, but instantly:

1. **Bound** — roofline: you *cannot* beat $\min(\text{compute},\ \text{bandwidth})$, whatever the microarchitecture.
2. **Decompose** — the CPI stack / interval model: *which* term (branch, I-fetch, LLC-miss) dominates, so you know where a mm² buys performance.
3. **Size** — Little's law / queueing: how much concurrency hides a given latency, and where a shared resource saturates.

And they answer a fourth question the simulator cannot: **when do I even need the simulator?** The escalation rule from [Simulation_Methodology §2/§4](01_Simulation_Methodology.md) is that a fast model is sound *as long as the effect you are studying does not feed back into the instruction path and the contention is not itself the object of study.* Roofline is enough to rule a kernel memory-bound; you escalate to gem5 only when reordering, prefetch interaction, path effects, or the exact shape of a contention curve is the thing you are measuring (§7).

---

## 2. Roofline — the throughput bound (extends [DSE §2.3](../01_Performance_Modeling_and_DSE.md))

The basic model and the per-kernel GPU treatment are in the DSE page; here is what to add so the bound is used correctly.

Attainable performance is
$$P \;=\; \min\!\big(\pi,\ \beta \cdot I\big),$$
where $\pi$ = peak compute rate (FLOP/s), $\beta$ = peak bandwidth (byte/s), and $I$ = **arithmetic (operational) intensity** in FLOP/byte. The **ridge point**
$$I^{*} \;=\; \pi/\beta$$
is the intensity at which the two roofs meet: $I<I^{*}$ is bandwidth-bound (on the slanted $\beta I$ roof), $I>I^{*}$ is compute-bound (on the flat $\pi$ roof). Everything an architect adds to this:

- **Ceilings (sub-roofs).** The flat roof $\pi$ is only reachable with full in-core parallelism: drop FMA, SIMD/vectorization, or ILP and you sit on a **lower horizontal ceiling**. The slanted roof $\beta$ is only reachable with prefetching and NUMA-aware placement; without them you sit on a **lower diagonal ceiling**. The *gap between a kernel's measured point and the relevant sub-roof names the specific optimization it needs* — this is roofline's real diagnostic value, not the top roof.
- **Hierarchical / cache-aware roofline.** Measure $I$ against traffic *at each level* — a DRAM roofline ($I_\text{DRAM}=\text{FLOP}/\text{DRAM bytes}$), an L2 roofline, etc. A kernel can be DRAM-compute-bound but L2-bandwidth-bound; the level whose point is lowest is the true bottleneck. This is the analytical shadow of the memory hierarchy a simulator models level by level.
- **What roofline cannot see — the crucial caveat.** Roofline is a **throughput ceiling that silently assumes enough ILP/MLP to saturate the bottleneck resource.** It says nothing about whether you can *reach* the roof at your actual concurrency. A latency-bound kernel with too few outstanding misses lands far *below* the memory roof with no roofline explanation — because the missing piece is **Little's law** (§5): reaching the $\beta$ roof requires $\beta\times L$ bytes in flight. **Roofline sets the ceiling; interval (ILP) and Little (MLP) decide attainment.** Treating the roof as a prediction rather than a bound is the model's most common misuse.

---

## 3. The interval / mechanistic model — core CPI in closed form

This is the analytical dual of gem5's O3 timing model ([gem5 §3](02_gem5.md)) and the rigorous version of the CPI stack ([DSE §2.1](../01_Performance_Modeling_and_DSE.md)). **Interval analysis** (Karkhanis–Smith 2004; formalized by Eyerman, Eeckhout, Karkhanis & Smith 2009) observes that a balanced out-of-order core issues at close to its **dispatch width $D$** during steady intervals, and that performance is set by **miss events that punctuate those intervals** — each punches a characterizable hole in the issue rate. Summing the per-event CPI increments (superposition) reproduces full-simulation CPI to within a few percent.

The mechanistic total-cycle equation:

$$C_{\text{total}}=\underbrace{\frac{N}{D}}_{\text{steady issue}}+\underbrace{\frac{D-1}{2D}\big(m_{iL1}+m_{iL2}+m_{br}+m_{dL2}\big)}_{\text{issue-ramp after each refill}}+\underbrace{m_{iL1}c_{iL1}+m_{iL2}c_{L2}}_{\text{I-fetch misses}}+\underbrace{m_{br}\,(c_{dr}+c_{fe})}_{\text{branch mispredicts}}+\underbrace{m_{dL2}(W)\,c_{L2}}_{\text{long-latency loads}}$$

where $N$ = dynamic instruction count; $D$ = dispatch width; $m_{iL1},m_{iL2},m_{br},m_{dL2}$ = counts of L1/L2 I-cache misses, branch mispredicts, and last-level data misses; $c_{iL1},c_{L2}$ = the corresponding miss latencies; $c_{fe}$ = front-end pipeline depth; $c_{dr}$ = the **window-drain (branch-resolution) time**; and $m_{dL2}(W)$ = the count of long-latency load misses that *cannot* be overlapped within a window of $W$ in-flight instructions. Two structural insights fall out — the same two the O3 simulator would take a CPU-hour to show:

- **Front-end misses (branch mispredict, I-cache miss)** cost $c_{fe}+c_{dr}$. The load-bearing, non-obvious term is $c_{dr}$: **a branch mispredict costs *more* than the pipeline depth**, because the back end must first *drain* the wrong-path window before the correct path can ramp. A whiteboard "penalty = pipeline depth" understates it.
- **Back-end misses (long-latency LLC loads)** cost the memory latency $c_{L2}$ — but only for the misses that serialize. **Memory-level parallelism (MLP) overlaps independent misses**, so the effective count is $m_{dL2}(W)$, not the raw miss count: a burst of $k$ independent misses under one ROB-fill costs ~one latency, not $k$. This is why enlarging the window (or MSHRs) helps memory-bound code, and why the term depends on $W$.

Accuracy is **~7% mean vs a width-4 O3 simulation** — good enough to rank pipeline-depth, width, and cache choices analytically, then confirm the survivors in gem5. **gem5's O3 model is this equation made executable and roughly 10% more accurate**; the interval model is why you can reason about a core without running one.

---

## 4. Amdahl — and the contention term it omits (USL)

Amdahl ([DSE §2.2](../01_Performance_Modeling_and_DSE.md)) gives the parallel ceiling from a serial fraction $p_s$:
$$S(N)=\frac{1}{(1-p_s)+p_s/N}\ \xrightarrow{N\to\infty}\ \frac{1}{1-p_s}.$$
Its blind spot is that it assumes the parallel part scales *perfectly*: it has **no term for contention** (serialized access to a shared resource) or **coherency** (keeping shared data consistent). Real multicore silicon has both, and they bend the curve *down*. **Gunther's Universal Scalability Law (USL)** adds exactly those two terms:

$$C(N)=\frac{N}{1+\alpha\,(N-1)+\beta\,N(N-1)},$$

where $N$ = degree of parallelism (cores/threads), $\alpha$ = **contention/serialization** coefficient (the Amdahl-like serial cost), and $\beta$ = **coherency/crosstalk** coefficient (pairwise consistency cost, hence the $N(N-1)$ that grows as $O(N^2)$). Setting $\beta=0$ recovers Amdahl. The qualitative change $\beta$ introduces is decisive: with any $\beta>0$ the curve is **retrograde** — throughput rises, peaks, then **declines** with more cores — and the peak is at

$$N^{*}=\sqrt{\frac{1-\alpha}{\beta}}.$$

For silicon this is not academic: $\beta$ is the **coherence traffic and shared-cache/NoC crosstalk** ([ACE_and_CHI](../12_ACE_and_CHI.md), [Network_on_Chip](../13_Network_on_Chip.md)) that a naive Amdahl estimate misses, and it is *why* adding cores past $N^{*}$ makes a workload slower. When you need the actual coherence traffic that sets $\beta$, that is the escalation to gem5 **Ruby** ([gem5 §4](02_gem5.md)).

---

## 5. Little's law and M/M/1 — the backbone of every contention model

[Simulation_Methodology §7](01_Simulation_Methodology.md) states that every credible memory/NoC simulator is, at bottom, a structural realization of the queueing curve. This is that spine, made rigorous — the single most reused piece of analysis in the whole notebook.

**Little's law** (distribution-free, needs only stationarity):
$$L=\lambda\,W,$$
where $L$ = mean number of items in the system, $\lambda$ = arrival/throughput rate, $W$ = mean time in system. It has two readings an architect uses constantly:

- **Latency hiding / sizing concurrency** (the GPU-occupancy result of [DSE §9.1](../01_Performance_Modeling_and_DSE.md)): to keep a unit busy through a latency $L_\text{lat}$ at throughput $\lambda$, you need $L=\lambda L_\text{lat}$ operations *in flight*. Too few → the unit stalls with peak throughput untouched.
- **Sizing the memory system** (the same law, memory units): to sustain bandwidth $\beta$ at memory latency $L_\text{mem}$, you need $\beta\times L_\text{mem}$ bytes outstanding — the **latency–bandwidth product**. In requests: $\text{MSHRs}\gtrsim \beta L_\text{mem}/\text{line}$. **This is the exact quantity roofline (§2) assumes you have** when it draws you on the bandwidth roof; if your MLP is below it, you fall off the roof.

**M/M/1** (Poisson arrivals, exponential service, one server) puts numbers on saturation. With utilization $\rho=\lambda/\mu$ (offered rate over service rate $\mu$):

$$W=\frac{1}{\mu-\lambda}=\frac{1/\mu}{1-\rho},\qquad W_q=\frac{\rho}{\mu(1-\rho)},\qquad L=\frac{\rho}{1-\rho}.$$

The $\dfrac{1}{1-\rho}$ factor **is the knee**: latency is roughly flat at low load and runs to infinity as $\rho\to 1$. This is why memory and NoC latency explode near saturation, and why a simulator that reports latency *flat* with load has no real queue and is optimistic ([Simulation_Methodology §3](01_Simulation_Methodology.md)).

**Variability matters — M/G/1 (Pollaczek–Khinchine).** Real service times are not exponential, and the wait scales with their variance:
$$W_q=\frac{\rho}{1-\rho}\cdot\frac{1+C_v^2}{2}\cdot\frac{1}{\mu},$$
where $C_v$ = coefficient of variation of service time. For deterministic service ($C_v=0$, M/D/1) the queueing wait is **half** the M/M/1 value; for bursty service it is worse. This is the analytical reason **DRAM and NoC schedulers exist**: FR-FCFS row-buffer reordering and NoC arbitration *reduce effective service variance* ($C_v$), buying back queueing latency without adding bandwidth ([DDR_Controller](../10_DDR_Controller.md), [Network_on_Chip](../13_Network_on_Chip.md)). Any credible memory/interconnect model is a structural realization of these three equations; recognizing them is how you audit one.

---

## 6. LogCA / LogGP — the offload and communication cost model

Roofline and interval assume the work is already where it runs. The **"is it worth offloading to the accelerator, and at what data size?"** question needs a communication-aware model. The lineage is **LogP** (Culler et al. 1993: **L**atency, **o**verhead, **g**ap = $1/\text{bandwidth}$ for small messages, **P**rocessors) and **LogGP** (Alexandrov et al. 1995), which adds **G** = gap *per byte* for long messages, so a $k$-byte transfer costs $\approx o + (k-1)G + L + o$.

**LogCA** (Altaf & Wood, ISCA 2017) specializes this to host↔accelerator offload with five parameters: **L** (interface latency, host→accelerator), **o** (host setup overhead), **g** (granularity = offloaded bytes), **C** (computational index, host cycles/byte), and **A** (peak acceleration). The speedup as a function of granularity is

$$\text{Speedup}(g)=\frac{C\,g^{\beta}}{o+L+\dfrac{C\,g^{\beta}}{A}},$$

with $\beta$ the algorithm's complexity exponent ($\beta=1$ linear, $>1$ super-linear). The decisive result is the **break-even granularity** — the smallest offload that even breaks even —

$$g_1=\frac{A}{A-1}\left(\frac{o+L}{C}\right)^{1/\beta},$$

which is set by the **interface cost $(o+L)$**, and is *essentially independent of $A$* for large $A$. The architect's takeaway: **a faster accelerator does not lower the size at which offload pays — the interface does.** A 100× engine behind a high-overhead bus is worthless for small tiles; you must either amortize over large $g$ or cut $(o+L)$. This is the analytical dual of the accelerator/offload cost that shows up in [AHB_AXI_APB](../11_AHB_AXI_APB.md)/[ACE_and_CHI](../12_ACE_and_CHI.md) and in the NPU dataflow models of [DSE §10](../01_Performance_Modeling_and_DSE.md).

---

## 7. When analytical suffices, and when you must escalate

The discipline is the escalation rule of [Simulation_Methodology §2/§4](01_Simulation_Methodology.md): **use the fastest model that distinguishes the choices in front of you; go cycle-level only when contention, reordering, or path/timing-feedback *is* the effect you are measuring.**

| Question | Analytical tool | It suffices when… | Escalate to (why) |
|---|---|---|---|
| Compute- or memory-bound? | Roofline (§2) | there is ample ILP/MLP to reach the roof | gem5 Timing/O3 or a GPU sim — *low-concurrency / latency-bound* regimes fall off the roof |
| Which stall dominates core CPI? | Interval (§3) | miss events are roughly independent | gem5 **O3** — complex prefetch overlap, data-dependent paths, wrong-path pollution |
| Parallel ceiling / core count? | Amdahl + USL (§4) | $\alpha,\beta$ are stable and measurable | gem5 **Ruby** — you need the *actual* coherence traffic that sets $\beta$ |
| Memory/NoC saturation latency? | Little + M/M/1 (§5) | traffic is near-Poisson, one bottleneck | Ramulator/DRAMSim3, Garnet — bursty/structured traffic, scheduler reordering |
| Is offload worth it, at what size? | LogCA/LogGP (§6) | the interface is uncontended | gem5 **FS** — shared-interface contention, driver/OS cost |

The failure mode in both directions is real: **trusting an analytical *ceiling* as a *prediction*** (roofline attainment without checking MLP) over-promises; **cycle-simulating a question a closed form already answers** wastes a CPU-week and, worse, hides the mechanism the equation would have made obvious. The mature workflow is **prune analytically, then cycle-simulate the survivors** — the DSE recipe of [DSE §5](../01_Performance_Modeling_and_DSE.md).

---

## 8. Numbers to memorize

| Quantity | Value / form | Why it matters |
|---|---|---|
| Ridge point | $I^{*}=\pi/\beta$ | the compute- vs memory-bound boundary |
| Roofline attainment | needs $\beta\times L$ bytes in flight | roof is a bound, not a prediction (Little) |
| Interval CPI | $C=N/D+\sum m_x c_x$ | core performance in closed form |
| Interval accuracy | ~7% vs width-4 O3 | good enough to rank, then confirm |
| Branch penalty | $c_{fe}+c_{dr}$ (> pipeline depth) | the drain term whiteboards forget |
| Amdahl ceiling | $1/(1-p_s)$ | serial fraction dominates |
| USL peak | $N^{*}=\sqrt{(1-\alpha)/\beta}$ | where more cores make it *slower* |
| Little's law | $L=\lambda W$ | concurrency to hide any latency |
| M/M/1 knee | $W\sim \dfrac{1}{1-\rho}$ | latency explodes near saturation |
| M/G/1 variance | $W_q\propto \dfrac{1+C_v^2}{2}$ | why schedulers cut $C_v$, not just add BW |
| LogCA break-even | $g_1\propto (o+L)/C$, ~indep. of $A$ | offload pays by *interface*, not accelerator speed |

---

## Cross-references

- **Down the stack:** [Performance_Modeling_and_DSE](../01_Performance_Modeling_and_DSE.md) (the working kernels this page makes rigorous — CPI stack, Amdahl, roofline, occupancy), [OoO_Execution](../05_OoO_Execution.md) (the structures the interval model abstracts), [Memory](../09_Memory.md) / [DDR_Controller](../10_DDR_Controller.md) & [Network_on_Chip](../13_Network_on_Chip.md) (the queues M/M/1 abstracts; the schedulers that cut $C_v$), [ACE_and_CHI](../12_ACE_and_CHI.md) (the coherence traffic behind the USL $\beta$ term).
- **Up the stack:** [Simulation_Methodology](01_Simulation_Methodology.md) (the fidelity ladder these models top, and the escalation rule), [Full_Chip_Modeling](../02_Full_Chip_Modeling.md) (analytical leaf models composed into a chip).
- **Sibling (this folder):** [gem5](02_gem5.md) — the executable form of the interval (O3), queueing (Ruby/memory), and roofline (DSE) models here; escalate to it per §7.

---

## References

- S. Williams, A. Waterman, D. Patterson, "Roofline: An Insightful Visual Performance Model for Multicore Architectures," *CACM*, 2009 — [PDF](https://people.eecs.berkeley.edu/~kubitron/cs252/handouts/papers/RooflineVyNoYellow.pdf).
- T. Karkhanis, J. E. Smith, "A First-Order Superscalar Processor Model," *ISCA*, 2004 — [retrospective PDF](https://sites.coecis.cornell.edu/isca50retrospective/files/2023/06/KARKHANIS_2004_FIRST.pdf).
- S. Eyerman, L. Eeckhout, T. Karkhanis, J. E. Smith, "A Mechanistic Performance Model for Superscalar Out-of-Order Processors," *ACM TOCS*, 2009 — [ACM](https://dl.acm.org/doi/10.1145/1534909.1534910).
- G. Amdahl, "Validity of the Single-Processor Approach…," *AFIPS*, 1967.
- N. J. Gunther, *Guerrilla Capacity Planning* (Universal Scalability Law) — [perfdynamics.com](https://www.perfdynamics.com/Manifesto/USLscalability.html).
- J. D. C. Little, "A Proof for the Queuing Formula $L=\lambda W$," *Operations Research*, 1961; L. Kleinrock, *Queueing Systems*, 1975 (M/M/1, M/G/1 Pollaczek–Khinchine).
- M. S. B. Altaf, D. A. Wood, "LogCA: A High-Level Performance Model for Hardware Accelerators," *ISCA*, 2017 — [PDF](https://research.cs.wisc.edu/multifacet/papers/isca17_logca.pdf).
- A. Alexandrov et al., "LogGP: Incorporating Long Messages into the LogP Model," *SPAA*, 1995; D. Culler et al., "LogP: Towards a Realistic Model of Parallel Computation," *PPoPP*, 1993.
