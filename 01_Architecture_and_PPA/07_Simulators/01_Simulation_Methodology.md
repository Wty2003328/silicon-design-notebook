# Simulation Methodology — How Architectural Simulators Actually Work

> **Prerequisites:** [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the analytical leaf models a simulator makes executable), [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md) (the tool/fidelity ladder this page explains conceptually).
> **Hands off to:** the per-tool pages in this folder — gem5, DRAM simulators, GPU simulators, accelerator/DNN simulators, and analytical models.

---

## 0. Why this page exists

Every performance or power number in this notebook that is not measured on real silicon comes out of a *model that was run* — a simulator. An architect who quotes "IPC 2.1" or "achieved bandwidth 480 GB/s" without knowing **what kind of model produced it** cannot say whether the number is trustworthy to 5% or to 50%. This page is about the machinery: what a simulator abstracts, what it computes cycle by cycle, how a benchmark is turned into a statistic, and where the error comes from. The specific tools (gem5, DRAMSim3, GPGPU-Sim, NeuSim) are instances of the ideas here; each gets its own page.

The single most important habit: **before you believe a simulated number, ask which rung of the fidelity ladder it came from and what that rung is allowed to model.**

---

## 1. The three questions every simulator answers

A processor simulator is a program that reproduces, at some chosen fidelity, *what a target machine would do when it runs a workload*. It answers up to three separable questions, and simulators differ mainly in **which of the three they model and how faithfully**:

1. **Functional — "what is the result?"** Execute the instructions so that architectural state (registers, memory) ends up correct. This is what an ISA emulator (QEMU, gem5 "atomic", Spike for RISC-V) does. It gives you the *right answer* and the *dynamic instruction count*, but **no timing**.
2. **Timing — "how long does it take?"** Model the microarchitecture — pipelines, caches, branch predictor, memory system — so you can count *cycles*, not just instructions. This is where IPC/CPI comes from.
3. **Physical — "what does it cost?"** Overlay energy, power, area, temperature onto the timed activity. This is the McPAT / Accelergy / DRAMPower / HotSpot layer covered in [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md).

The clean architecture — used by gem5, GPGPU-Sim, and most modern tools — **separates functional from timing** (the "execute-in-execute" or "timing-directed" split): a functional core guarantees correctness while a timing model, driven by the same instruction stream, accounts for when each event happens. Keeping them separate is what lets one functional model feed many timing models.

---

## 2. The fidelity–speed tradeoff (the master curve)

Simulation fidelity and simulation speed trade off across roughly six orders of magnitude. Knowing where a tool sits tells you both its error bar and its cost.

| Level | What it models | Typical slowdown (host/target) | Error on cycles | Example |
|---|---|---|---|---|
| Native / functional emulation | correctness only, no time | 1–10× (JIT); ~10²–10³× (interp.) | n/a (no timing) | QEMU, Spike, gem5-atomic |
| Analytical / mechanistic | closed-form CPI from a few parameters | ~instant | 5–20% | interval model, roofline |
| Instruction-timing | one timing event per instruction, coarse pipe | ~10³–10⁴× | 10–20% | simple in-order timing |
| Event-driven cycle-approximate | components + queueing, not every latch | 1k–10k× | 5–15% | gem5-O3, GPGPU-Sim, Ramulator |
| Cycle-accurate (µarch) | every pipeline stage, validated to a real core | 10k–100k× | <5% vs that core | vendor-internal, some gem5 configs |
| RTL simulation | the actual gates/registers | 10⁶–10⁷× | exact (it *is* the design) | Verilator/VCS on the RTL |
| Emulation / FPGA prototype | RTL in hardware | 10–1000× | exact, fast | Palladium, FireSim |

Two lessons. First, **"cycle-accurate" is a claim about a specific target**, not a universal virtue — an O3 model validated against a Cortex-A76 is not cycle-accurate for a Neoverse-V2. Second, the reason architects live mostly on the middle two rungs (analytical + event-driven) is the **speed budget**: a design-space exploration (DSE) sweep of thousands of configurations cannot afford RTL, so you push physics down into pre-characterized per-event costs (CACTI energies, DRAM timings) and keep the *composition* fast — the theme of [Full_Chip_Modeling §2](../01_Modeling/02_Full_Chip_Modeling.md).

### 2.1 Where the slowdown comes from — the ladder quantified

The six orders of magnitude are not folklore; they fall out of a one-line accounting identity. A simulator advances simulated time by executing *host* instructions, so its cost is set by **how many host instructions it must run to move the target forward by one cycle**. Call that the *work per simulated cycle* $w$. It factors exactly as the brief's product:

$$w \;=\; \underbrace{i_{\text{ev}}}_{\substack{\text{host instr}\\\text{per event}}}\;\times\;\underbrace{n_{\text{ev}}}_{\substack{\text{events per}\\\text{simulated cycle}}},$$

where an *event* is a modeled state transition (a stage advance, a wakeup, a cache probe, a net toggle). The host retires host instructions at rate $R_h$ (instr/s), so simulating one target cycle costs $w/R_h$ host-seconds while representing $1/f$ target-seconds ($f$ = target clock). The **slowdown** is the ratio of the two, and the **simulated throughput** $\Theta$ (simulated cycles per host-second) is its reciprocal scaled by $f$:

$$\boxed{\,S \;=\; \frac{w/R_h}{1/f} \;=\; \frac{w\,f}{R_h}\,}, \qquad \Theta \;=\; \frac{f}{S} \;=\; \frac{R_h}{w},$$

where $S$ = wall-clock/target-time slowdown, $R_h$ = host retirement rate (instr/s), $f$ = target clock (Hz), $w$ = host instructions per simulated cycle. When host and target run at comparable instruction rates ($R_h\approx f$, e.g. a 3 GHz host at IPC≈1 modeling a 3 GHz core) the identity collapses to the brief's approximation $S\approx w$ — **the slowdown *is* the work-per-simulated-cycle**. Everything below just reads $w$ off each rung.

Take an illustrative modern host at $R_h\approx 3\times10^9$ host-instr/s. The rungs differ only in $w$ — how much bookkeeping each simulated cycle demands:

| Rung | $w$ (host instr / sim cycle or instr) | $\Theta=R_h/w$ | Slowdown $S$ @ 3 GHz |
|---|---|---|---|
| Functional, JIT/binary-translation | $\sim$1–10 /instr | $10^8$–$10^9$ (100s MIPS–GIPS) | 1–10× |
| Functional, interpreted (gem5-atomic) | $\sim10^2$–$10^3$ /instr | $\sim10^6$ (**~MIPS**) | $10^2$–$10^3$× |
| Instruction-timing | $\sim10^3$–$10^4$ | $\sim3\times10^5$ (**~100s KIPS**) | $10^3$–$10^4$× |
| Cycle-accurate (every structure/cycle) | $\sim10^4$–$10^6$ | $\sim3\times10^4$ (**~10s KIPS**) | $10^4$–$10^6$× |
| RTL / gate (every toggling net/cycle) | $\sim10^6$–$10^8$ | $1$–$10^3$ (**~Hz–KHz**) | $10^6$–$10^7$× |

The progression is mechanical: functional does $O(1)$ host work per *instruction* and skips cycles entirely (no $n_{\text{ev}}$); instruction-timing adds one timing event per instruction; cycle-accurate walks *every* modeled structure *every* cycle ($n_{\text{ev}}$ = hundreds of components), so $w$ jumps ~10–100×; RTL evaluates every net that toggles ($n_{\text{ev}}\sim10^6$–$10^8$ for a full chip). This is why **the cycle-level rungs span $10^3$–$10^6\times$ real time** — the low end a lean instruction-timing or cycle-approximate core ($w\sim10^3$–$10^4$), the high end a full cycle-accurate core with a detailed memory system and every latch modeled ($w\sim10^5$–$10^6$); the span *is* how much of the machine each cycle touches.

**Worked number — wall-clock to simulate 1 s of a 3 GHz chip.** One target-second is $C=f\cdot1\text{ s}=3\times10^9$ cycles, so wall-clock $=C/\Theta$:

| Rung | $\Theta$ | Wall-clock for 1 s of target |
|---|---|---|
| Functional (JIT) | $10^9$ instr/s | $3\times10^9/10^9 \approx \mathbf{3\ s}$ |
| Functional (interp.) | $\sim3\times10^6$ | $\sim10^3\ \text{s} \approx \mathbf{17\ min}$ |
| Instruction-timing | $\sim3\times10^5$ | $10^4\ \text{s} \approx \mathbf{2.8\ hours}$ |
| Cycle-accurate | $\sim3\times10^4$ | $10^5\ \text{s} \approx \mathbf{1.2\ days}$ |
| RTL (100 Hz) | $\sim10^2$ | $3\times10^7\ \text{s} \approx \mathbf{347\ days}$ |

Cycle-accurate turns *one second* of chip life into *a day* of compute; RTL turns it into *a year*. And a real benchmark is not one target-second — a SPEC run is $\sim10^{12}$ instructions $\approx$ hundreds of target-seconds, so at $3\times10^4$ instr/s it is $\sim10^{12}/3\times10^4 \approx 3\times10^7$ s $\approx$ **a CPU-year per benchmark**. That single figure is the entire justification for sampling (§5): you physically cannot run the whole workload at the fidelity you need, so you must run a *provably representative slice* of it.

---

## 3. The event-driven engine — the heart of a timing simulator

Cycle-approximate and cycle-accurate timing simulators are almost always **discrete-event simulators**. The core is not "loop over cycles and update everything" (that is a cycle-*stepped* engine, simpler but slower); it is a **priority queue of events keyed by the tick they fire**:

```
event_queue = min-heap keyed by tick
while not empty and tick < end:
    (tick, event) = pop-min          # jump straight to the next thing that happens
    event.process()                  # may schedule new events at tick + latency
```

The key idea is **time skipping**: if nothing happens between cycle 100 and cycle 137 (a load is in flight to DRAM), the engine jumps directly to 137 instead of simulating 37 empty cycles. Latencies are expressed as *"schedule this event `+N` ticks from now"* — a cache miss schedules its fill event at `now + t_DRAM`; a pipeline stage schedules the next stage at `now + 1`. This is why a memory-bound workload can simulate *faster* than a compute-bound one of the same length: fewer events per unit of simulated time.

Consequences an architect should internalize:

- **Latency is data, not control flow.** Every modeled latency (cache hit time, DRAM `t_RCD`, NoC hop, NVLink transfer) is a `+N` on a scheduled event. Changing a latency parameter is changing an addend, which is why sensitivity sweeps are cheap and why a wrong latency constant silently biases every downstream number.
- **Contention emerges from shared event resources.** Two requests contend not because the model has an explicit "contention formula" but because they compete for the same port/bank/queue whose events serialize. A model with no shared resource *cannot* produce queueing delay — it will report the sum of standalone latencies and be optimistic (the auditor's first probe, [Full_Chip_Modeling §5.1](../01_Modeling/02_Full_Chip_Modeling.md)).
- **Determinism and reproducibility** come from a fixed tie-breaking rule for events at the same tick. Non-determinism in parallel simulators (multiple event queues) is a real correctness hazard.

**Why event-driven beats cycle-driven — the cost, derived.** Compare the two engines by counting operations. A **cycle-stepped** engine, over $C$ simulated cycles with $M$ modeled components, visits every component every cycle: cost $\Theta(C\cdot M)$ — set by the machine's *size*, independent of what actually happened. An **event-driven** engine touches only the $E$ events that actually fire; each `pop-min` / `push` on a binary-heap priority queue costs $O(\log \bar n)$, where $\bar n$ is the live queue occupancy (outstanding, not-yet-fired events). Total cost $\Theta(E\log\bar n)$ — set by the machine's *activity*. The ratio is

$$\frac{\text{event-driven}}{\text{cycle-driven}} \;=\; \frac{E\log\bar n}{C\,M} \;=\; a\log\bar n, \qquad a\equiv\frac{E}{C\,M}\ (\text{activity density}),$$

where $a$ = fraction of components that generate an event in an average cycle. Event-driven wins whenever $a\log\bar n<1$, i.e. $a<1/\log\bar n$ — **whenever activity is sparse.** The intuition the algebra proves: cycle-stepping pays for *capacity* ($C\!\times\!M$ latch updates whether or not anything moved); event-driven pays for *events*, and a mostly-idle chip has far fewer events than it has (components × cycles). When a load is in flight to DRAM for 300 cycles, cycle-stepping still runs $300M$ updates; the event engine schedules one fill event at `now + t_DRAM` and **time-skips** the 300 empty cycles — the skip grows without bound as latency grows, which is why a memory-bound trace can simulate *faster* than a compute-bound one of equal length.

*Worked number.* A core with $M=10^3$ modeled sub-structures (ports, FUs, queue slots, MSHRs, cache banks…) over $C=10^6$ cycles costs the cycle-stepped engine $C\,M=10^9$ updates. Suppose the workload averages $E=3\times10^6$ events (≈3 active structures per cycle — realistic for an IPC≈1–2 core where only a handful of units do anything each cycle), so $a=3\times10^6/10^9=3\times10^{-3}$. With heap occupancy $\bar n\approx128$, $\log_2\bar n=7$, event-driven cost $=E\log_2\bar n=2.1\times10^7$ heap-ops — a $\mathbf{48\times}$ win ($a\log_2\bar n=0.021$). The win *is* the sparsity: were the model dense ($a\to1$, every structure every cycle) event-driven would *lose* by the $\log\bar n$ factor plus its per-event overhead, which is exactly why small, fully-active models (and some RTL) are cycle-stepped instead.

**The time-ordered execution invariant — why the heap is correct, not just fast.** The engine must process events in nondecreasing timestamp order, and the min-heap enforces exactly that. It is a *correctness* requirement, provable by one premise: **no event schedules a consequence in the past** — every modeled latency is $\ge 0$, so an event firing at tick $t$ can only create events at ticks $t'\ge t$. *Claim:* popping the minimum tick each step processes events in true temporal order. *Proof (induction on pops).* Suppose every event popped so far had tick $\le t$, and we now pop tick $t$ as the heap minimum. Any not-yet-processed event either is already in the heap (tick $\ge t$, since $t$ is the min) or has not been created — and it can only be created by an event at tick $\ge t$, hence stamped $\ge t$. So no event with tick $<t$ can ever appear after we pass $t$: the pop order equals the temporal order. $\square$ The payoff is that when an event fires, *every* input that could causally affect it (everything earlier) is already resolved — the model never reads stale state. The premise is load-bearing: zero-latency combinational chains within a tick would violate "$\ge 0$ strictly," which is why engines add a **delta/priority sub-order within a tick** (the same tie-break that gives determinism), sequencing zero-time updates without breaking causality.

**Contention as an emergent output — quantified.** Because a shared resource is modeled as a stream of *service events* (a port grants one request per cycle; a bank accepts one access per $t_{RC}$), two requests for it cannot both be served at once: the arbiter schedules the second's grant at a later tick, and the gap between arrival and grant *is* the queueing delay — no "contention formula" is written anywhere. That emergent delay obeys the queueing law of §7 because the event schedule is literally a sample path of the queue. *Worked number:* a port of service rate $\mu=1$/cycle under offered load $\lambda=0.8$/cycle runs at utilization $\rho=\lambda/\mu=0.8$; the M/M/1 mean wait is $W_q=\rho/(1-\rho)=0.8/0.2=\mathbf{4\ \text{cycles}}$ (residence $W=1/(1-\rho)=5$) — four cycles of delay that the modeler never typed, produced purely by serializing grant events at a finite-rate resource. A model with *no* shared-resource event (fixed-latency memory, §7) has $\rho$ nowhere and reports the sum of standalone latencies: it is structurally incapable of showing this delay and is therefore optimistic (the auditor's first probe, [Full_Chip_Modeling §5.1](../01_Modeling/02_Full_Chip_Modeling.md)).

---

## 4. Trace-driven vs execution-driven — where the instruction stream comes from

A timing model needs a stream of operations to time. There are two ways to feed it, and the choice determines what the simulator can and cannot capture.

**Execution-driven.** The functional model runs the real program and hands each instruction to the timing model as it goes. This is the gold standard because it captures **path effects**: mispredicted-branch wrong-path instructions that pollute caches, data-dependent control flow, spin-loops that change with timing, and the feedback where timing affects which instructions even execute (e.g., a lock acquired in a different order). gem5, GPGPU-Sim, and Accel-Sim in execution mode work this way.

**Trace-driven.** You record a trace once (instructions, addresses) and replay it into the timing model. Much faster and repeatable, and it decouples workload capture from model iteration — but it **freezes the path**. A trace taken on one machine cannot show wrong-path pollution for a *different* branch predictor, and it mishandles anything where timing changes the instruction stream (multithreaded races, most spin-waiting). Trace-driven is excellent for memory-system and NoC studies (addresses are what matter) and dangerous for core-microarchitecture studies (path matters). Accel-Sim's trace mode and most DRAM-simulator front-ends are trace-driven for exactly this reason: for a DRAM channel, the *address/time stream* is a faithful stimulus, but for an OoO core the path is the point.

The rule: **trace-driven is sound when the thing you are studying does not feed back into the instruction path; execution-driven is required when it does.**

**The path-dependence limitation, derived.** Write the executed instruction stream as a function of the microarchitecture, $\Pi(\mu)$ — the sequence of *fetched* operations (committed **and** speculative) that machine $\mu$ actually runs. A trace is a *frozen* $\Pi(\mu_0)$, captured under some reference machine $\mu_0$ (or under no timing at all). Replaying it into a timing model of $\mu_1$ evaluates the timing of $\mu_1$ **on the path of $\mu_0$** — which is correct **iff $\Pi(\mu_1)=\Pi(\mu_0)$**, i.e. iff the path does not depend on the microarchitecture. That premise splits cleanly:

- **Committed data-flow path** (single thread, fixed inputs): branch *outcomes* are data-determined, so the committed instruction and address stream is $\mu$-independent — $\Pi_{\text{commit}}(\mu_1)=\Pi_{\text{commit}}(\mu_0)$. A trace is *faithful* here. This is why DRAM/NoC and cache-capacity studies are safely trace-driven: for a memory channel the committed address/time stream is a sound stimulus.
- **Speculative (wrong-path) instructions** depend on the *predictor*, which is part of $\mu$: $\Pi_{\text{spec}}(\mu_1)\ne\Pi_{\text{spec}}(\mu_0)$. A committed-only trace contains *none* of them; a trace *with* wrong-path is welded to $\mu_0$'s predictor. Either way the timing model of $\mu_1$ cannot reproduce its own wrong-path cache pollution, prefetching, or port contention.
- **Timing-fed-back path** (multithread races, lock-acquire order, spin-loop iteration counts, adaptive code): here $\Pi$ depends on *relative timing*, so a trace recorded at one timing is simply a *different execution* of a different machine.

**Quantify the regime where it matters.** The error is carried by the missing (or wrong) wrong-path instructions, whose volume is a counting quantity. With branch fraction $b$, predictor accuracy $a$, and $W_{\text{wp}}$ wrong-path instructions fetched per misprediction (≈ resolution latency × fetch width, capped by the ROB depth), the wrong-path fetch overhead per committed instruction is

$$\phi_{\text{wp}} \;=\; b\,(1-a)\,W_{\text{wp}},$$

where $b$ = branches/instr, $(1-a)$ = mispredict probability, $W_{\text{wp}}$ = wrong-path instr per mispredict. *Worked number:* $b=0.2$, $a=0.95$, $W_{\text{wp}}=30$ (≈15-cycle resolution × 2-wide fetch) give $\phi_{\text{wp}}=0.2\cdot0.05\cdot30=\mathbf{0.30}$ — **30% extra fetched instructions live on wrong paths**, every one probing the I-cache/BP and, for wrong-path loads, the D-cache. A committed-only trace omits all of it. The decisive feature is the $(1-a)$ scaling: for a *predictable* core study ($a\to0.99$, $\phi_{\text{wp}}\!\to\!0.06$) the distortion on the timing number is small (~1–2%), but for a *mispredict-heavy* workload ($a\approx0.90$, $\phi_{\text{wp}}\!\approx\!0.6$) ignoring wrong-path can bias IPC/miss-rate by ~10–15%. So trace-driven core-µarch error $\propto (1-a)$ — negligible on branchy-but-predictable code, disqualifying on hard-to-predict code — which is the precise boundary the qualitative rule above draws.

---

## 5. How a workload becomes a number — the part people get wrong

Running "the benchmark" end-to-end at cycle-approximate fidelity is usually infeasible (a 1-trillion-instruction SPEC run at 10 000× slowdown is months of wall-clock). So the reported number is almost always the product of a **sampling and warm-up methodology**, and most simulation mistakes live here, not in the model.

1. **Region of interest (ROI).** You skip initialization and measure only the steady-state region — often via magic instructions / `m5_work_begin` markers. Reporting whole-program numbers when the ROI is a kernel is a classic error.
2. **Warm-up.** When you fast-forward (functional-only) to the ROI, the caches, TLBs, and branch predictor are *cold*. Measuring immediately overstates miss rates. You must run a **warm-up window** (functionally warming caches, or detailed-warming for a few million instructions) before you start counting. Under-warming is the most common silent bias in student and paper results.
3. **Sampling.** Instead of all N instructions, measure representative slices:
   - **SimPoint** clusters the program's execution into phases using basic-block vectors (BBVs) and simulates one representative *simpoint* per phase, then weights them. It exploits the fact that program behavior is *phasic and recurrent*.
   - **SMARTS / statistical sampling** takes many small, uniformly spaced detailed samples with functional warming between them and reports a mean with a **confidence interval** — the number comes with $\pm$ error and an $n$, like any Monte-Carlo estimate.
4. **The statistic.** Cycles are accumulated over the measured region; then

$$\text{CPI} = \frac{\text{cycles}}{\text{instructions}}, \qquad \text{IPC} = \frac{1}{\text{CPI}}, \qquad \text{achieved BW} = \frac{\text{bytes moved}}{\text{cycles}/f}.$$

A subtle point: **speedups must be summarized with the geometric mean** across benchmarks (ratios), while rates at a fixed workload use the arithmetic mean — mixing these is a real reporting bug.

The auditor's checklist for any simulated result: *What was the ROI? How long was warm-up? What sampling method and how many samples? Is there a confidence interval? Geometric or arithmetic mean?* A number without these is a number without an error bar.

### 5.1 The sampling error bound — why ~1% of the program suffices

Statistical sampling is not a heuristic; it comes with a *provable* error bar, and the proof is the reason you can trust a CPI computed from 1% of a program. Partition the run into $K$ equal intervals (SMARTS uses $\sim\!1000$-instruction units); interval $i$ has a local CPI $x_i$, and the true whole-program CPI is the mean $\mu=\frac1K\sum_{i=1}^K x_i$. Sample $n\ll K$ intervals uniformly at random and estimate with the sample mean $\bar x_n=\frac1n\sum_j x_{i_j}$.

**Derivation (CLT → standard error).** The estimator is unbiased, $\mathbb E[\bar x_n]=\mu$, and for i.i.d.-enough draws the Central Limit Theorem gives $\bar x_n\approx\mathcal N(\mu,\ \sigma^2/n)$ with $\sigma^2=\operatorname{Var}(x_i)$ the *inter-interval* CPI variance. So the standard error shrinks as the square root of the sample count,

$$\text{SE}(\bar x_n)=\frac{\sigma}{\sqrt n},$$

and a $100(1-\delta)\%$ confidence interval is $\bar x_n\pm z_{\delta/2}\,\sigma/\sqrt n$ (with $z_{0.025}=1.96$ for 95%). (Sampling *without* replacement adds the finite-population factor $\sqrt{1-n/K}\approx1$ for $n/K\sim1\%$ — negligible, which is itself why 1% coverage is the sweet spot.) This is precisely why SMARTS reports "a mean with a confidence interval": the $\pm$ is $z\sigma/\sqrt n$, computed, not hoped.

**Solving for the sample count.** Demand the half-width be a fraction $\varepsilon$ of the mean, $z\,\sigma/\sqrt n\le\varepsilon\mu$, and invert:

$$\boxed{\,n\;\ge\;\Big(\frac{z\,\sigma}{\varepsilon\,\mu}\Big)^{2}\;=\;\Big(\frac{z\,c_v}{\varepsilon}\Big)^{2}\,},\qquad c_v\equiv\frac{\sigma}{\mu}\ (\text{coefficient of variation}),$$

so $n\propto(\sigma/\varepsilon)^2$ — the brief's law. The quadratic is the whole economics of sampling: **halving the error bar costs 4× the samples**, so you stop at "good enough," never chase a tiny CI.

**Worked number — ±2% CPI at 95% confidence.** Take $z=1.96$, $\varepsilon=0.02$, and an illustrative $c_v=1.0$ (CPI standard deviation ≈ its mean — heavy phase variation between compute-bound and memory-bound stretches is normal):

$$n\ge\Big(\frac{1.96\times1.0}{0.02}\Big)^2=(98)^2=9604\approx\mathbf{10^4\ \text{intervals}}.$$

At ~$10^3$–$10^4$ instructions per interval that is $10^7$–$10^8$ instructions *detailed-simulated*; against a $10^{10}$-instruction program that is $0.1$–$1\%$ coverage — the "~1% of instructions" of the fidelity-ladder folklore, now with a *number and a confidence*. The dependence on $c_v^2$ is the sensitivity: a placid $c_v=0.5$ needs only $n\approx(49)^2\approx2400$; a spiky $c_v=1.5$ needs $n\approx(147)^2\approx2.2\times10^4$. SMARTS's canonical "$\sim\!10{,}000$ systematic samples" lands exactly in this band — the derivation *is* its design rule. And this is why cheap analytical pruning precedes expensive simulation ([Performance_Modeling_and_DSE §5](../01_Modeling/01_Performance_Modeling_and_DSE.md)): you spend the $10^4$-sample budget only on survivors.

**The systematic bias $\sqrt n$ cannot fix.** The CLT bound governs only the *random* error. Cold-start is a *bias*: when you fast-forward to a sample, the caches/TLB/predictor hold the wrong state, so every sampled interval is charged too many misses and its CPI reads high by a systematic offset $\beta_{\text{bias}}$. The estimator then converges to $\mu+\beta_{\text{bias}}$, and the mean-squared error decomposes as

$$\text{MSE}=\underbrace{\beta_{\text{bias}}^{\,2}}_{\text{systematic (floor)}}+\underbrace{\sigma^2/n}_{\text{random}\;\to\;0}.$$

As $n\to\infty$ the variance vanishes but the bias² **remains** — you cannot sample your way out of a warm-up error; the RMSE floors at $|\beta_{\text{bias}}|$. Under-warming is thus the most common *silent* bias (it does not widen the CI, it moves its center), which is why the auditor asks warm-up length before sample count.

**How warming bounds the bias.** The offset is inherited from state whose *reuse distance* exceeds what the sample restored, so two remedies attack two timescales. **Functional warming** (cheap, no timing) keeps long-lived state — LLC, TLB, branch-predictor tables — continuously correct between samples, removing the bias for structures whose contents outlive an interval. **Detailed warming** runs the timing model for a window $w$ before each measured window to also correct short-lived state (pipeline fill, MSHR/ROB occupancy, near-LRU order). The residual bias decays roughly geometrically in $w$, $\beta_{\text{bias}}\sim\beta_0\,e^{-w/\tau}$, where $\tau$ is the structure's fill time (instructions to reach steady state); choosing $w\gtrsim$ a few $\tau$ drives the bias below the random SE, at which point the CLT confidence interval is honest again. *Worked number — why big state needs functional, not detailed, warming:* a 2 MB LLC has $2^{21}/64=3.3\times10^4$ lines; at an L2-fill miss rate of $5\times10^{-3}$/instr its cold-fill time is $\tau\approx3.3\times10^4/5\times10^{-3}\approx\mathbf{6.6\times10^6\ \text{instructions}}$. Detailed-warming *millions* of instructions before every one of $10^4$ samples is unaffordable — so SMARTS functional-warms the LLC/TLB/BP for the *entire* run (cheap) and reserves detailed warming for the short-lived state ($w\sim2000$ instr per sample), pushing $\beta_{\text{bias}}$ under the ±2% SE. The confidence interval you quote is only as honest as this bias is small.

---

## 6. How compute is modeled — the timing of a core

A timing core model does not re-simulate the ALU's logic; it models the **structural and hazard constraints that decide when an instruction can advance**. Two canonical fidelities:

- **In-order** model: an instruction issues when its operands are ready and its functional unit is free; a miss/hazard stalls the pipe. Cheap; adequate for simple cores and for first-order studies.
- **Out-of-order (O3)** model: the machine that matters for big cores. The simulator maintains the *structures that impose the limits* — a reorder buffer (ROB) bounding how far ahead retirement can lag, physical registers and a rename map, reservation-station/IQ slots, load/store queues, and issue-port counts. Each cycle it (conceptually) fetches up to width, renames against free physical registers, wakes up ready instructions, selects up to issue-width for execution, and retires in order from the ROB head. **The point of the model is that throughput is set by whichever structure saturates first** — you learn *why* IPC is 1.8 not 4 (ROB-full? LSQ-full? a long dependence chain? issue-port pressure?), which is exactly the conceptual "what is stored and why" framing this notebook prefers over signal lists. See [OoO_Execution](../02_CPU/03_OoO_Execution.md) for the structures themselves.

The **interval / mechanistic model** (Karkhanis–Smith) is the analytical dual: instead of simulating the window, it says a balanced OoO core issues at its ideal width *between* "miss events" (branch mispredicts, last-level-cache misses), and each miss event punches a hole in the issue rate of a characterizable width. CPI becomes a base term plus $\sum(\text{miss rate}\times\text{penalty})$ — closed-form, instant, and good to ~10–20%. It is why you can reason about a core on a whiteboard; the event-driven O3 model is the same story made executable and 10% more accurate.

**The cost-model abstraction — what "modeling compute" reduces to.** A timing core never computes an operation's *result* (the functional model already did) — it assigns each operation a **time and a resource occupancy**, and nothing more. Ask what the model must answer to schedule one instruction and the minimal record falls out (a ROB-style "it must do these jobs, therefore it must hold X" argument): (1) *when is the result available to dependents?* → a **latency** $L$; (2) *when can the unit accept the next op?* → an **initiation interval** $II$ (the reciprocal throughput); (3) *how many can run at once?* → a **port/unit count** $P$. Two numbers per op-class plus a width is *minimal sufficient*: a single "latency" cannot express a pipelined unit ($L>1$ but $II=1$), and a single "throughput" cannot express a dependence stall — so neither alone suffices, and $(L,II,P)$ together do. In the event engine this is literally how compute time is assigned: "issue op" schedules "result-ready" at `now + L` and marks the port busy until `now + II`; the dependence graph does the rest.

*Worked number — the same eight ops, two times.* A pipelined FP adder with $L=4$ cycles, $II=1$. A **dependent** chain of $k$ adds (each needs the previous result) is latency-bound at $k\cdot L$; $k=8\Rightarrow32$ cycles. The **same eight** adds with no dependences are throughput-bound at $k\cdot II + L=8+4=12$ cycles. One functional unit, one op-count, **2.7× apart** — the gap is entirely the dependence structure the event schedule encodes through $(L,II)$, which is exactly what the interval model summarizes and the O3 model measures. This is why the cost model is $(L,II,P)$ and not a single "op takes $t$ cycles": the scalar throws away the distinction that decides the answer.

---

## 7. How the memory system is modeled — usually the dominant term

For most modern workloads the memory system decides performance, so its model deserves the most scrutiny. Fidelity rungs:

- **Fixed-latency** ("simple/classic" memory): every access costs a constant $t$. Fast, and wrong whenever the memory system is contended — it has *no queueing term*, so it cannot show bandwidth saturation.
- **Cache hierarchy model**: per-level capacity/associativity/latency, MSHRs bounding outstanding misses, coherence (the gem5 Ruby model is a full coherence-protocol state machine). This produces the miss-rate and MLP effects that fixed-latency cannot.
- **Cycle-level DRAM model** (Ramulator, DRAMSim3): models banks/ranks/channels, the JEDEC timing constraints ($t_{RCD}, t_{RP}, t_{RAS}, t_{FAW}, t_{WTR}, \dots$), the row-buffer, refresh, and the **scheduler** (FR-FCFS: first-ready, first-come-first-served, which reorders to hit open rows). Achieved bandwidth is an *output* of this contention, not an input — a memory-bound number that came from a fixed-latency model is not credible.

The theoretical backbone across all of it is **queueing**: a shared resource of service rate $\mu$ under offered load $\lambda$ has utilization $\rho=\lambda/\mu$, and mean latency grows as $\sim 1/(1-\rho)$ — latency runs to the knee as you approach saturation. Every credible memory/NoC simulator is, at bottom, a structural realization of that curve; a model that reports latency flat with load has no queue and is optimistic. (DRAM device/controller detail: [Memory](../03_Memory/03_Memory.md), [DDR_Controller](../03_Memory/04_DDR_Controller.md).)

**Where the $1/(1-\rho)$ knee comes from — and why bandwidth is an *output*.** How the memory cost model assigns time per access is the mirror of the compute one (§6): each request holds a server (bank/channel) for a service time, and the *delay* is service plus the wait for the server to free. Model one shared channel as an M/M/1 queue: with Poisson arrivals at rate $\lambda$ and exponential service at rate $\mu$, the mean residence time is

$$W=\frac{1}{\mu-\lambda}=\frac{1/\mu}{1-\rho},\qquad \rho=\frac{\lambda}{\mu},$$

so the *observed* per-access latency is the bare service time $1/\mu$ amplified by $1/(1-\rho)$ — flat at low load, diverging at the saturation knee. The modeler never writes a "latency vs load" curve; it *emerges* from serializing access events at a finite-rate server (the §3 mechanism). *Corollary — achieved bandwidth is not a parameter.* A bank can begin a new access only every row-cycle $t_{RC}$ on a row-buffer miss (activate → read → precharge), so its throughput ceiling is $1/t_{RC}$ accesses, and $B$ banks deliver at most $B\cdot\text{burst}/t_{RC}$ bytes/s; **achieved bandwidth $=\min(\text{demand},\ \text{this structural ceiling})$**, an output of the event schedule, not an input. *Worked number:* with a 64 B burst, $t_{RC}=45$ ns, and a pathological all-row-miss stream (row-buffer thrash), one bank sustains $64\text{ B}/45\text{ ns}\approx1.4$ GB/s and 16 banks $\approx\mathbf{23\ \text{GB/s}}$ — a small fraction of the datasheet peak, *purely because the access pattern missed the row buffer*. The FR-FCFS scheduler exists to raise the row-hit rate and pull achieved BW back up; a fixed-latency model has no $t_{RC}$, no banks, no $\rho$ — it cannot produce any of this and will quote the peak, which is why "a memory-bound number from a fixed-latency model is not credible."

---

## 8. Validation, calibration, and the error budget

A simulator is a *hypothesis about a machine*; it is only as good as its validation.

- **Validation** compares simulated statistics against real hardware performance counters on the same binaries. The honest figure of merit is not a single average error but the **error distribution and the correlation of trends** — a model that is 15% off in absolute IPC but ranks configurations correctly is often *more useful for DSE* than one that is 5% off but mis-orders two designs.
- **Calibration** fits free parameters (latencies, penalties, McPAT's technology knobs) to close the gap; an *un-calibrated* architectural model can be 2× off, a calibrated one ~20–30% (see the [Full_Chip_Modeling §1.7 accuracy ladder](../01_Modeling/02_Full_Chip_Modeling.md)).
- **Composed error**: when you chain models (perf → power → thermal), errors compound. A ±20% power model feeding a thermal model feeding a DVFS governor can swing the final throughput materially — which is why the perf/power/thermal loop must be *co-validated*, not validated stage by stage in isolation.

**Why trends survive calibration error — a monotonicity theorem.** The claim that a biased model still ranks correctly is provable, and it turns on *which kind* of error you have. Split a model's error into a **systematic** part (a consistent distortion $\hat y=g(y)$) and a **random** part (SD $s$ per estimate). *Systematic error cancels in ranking:* if $g$ is monotone increasing then $\hat y_A>\hat y_B\iff y_A>y_B$ for every pair — order is preserved under *any* monotone distortion, and for a pure multiplicative bias $\hat y=y(1+\beta)$ the ratio $\hat y_A/\hat y_B=y_A/y_B$ is preserved *exactly*, so a uniformly-15%-high model has **zero** ranking error. That is the theorem behind "use the model for relative comparisons." *Random error sets the resolution limit:* two configs a true $\Delta$ apart give a difference $\hat y_A-\hat y_B$ with SD $s\sqrt2$, distinguishable at 95% only if

$$\Delta \;>\; 1.96\,s\sqrt2 \;\approx\; 2.8\,s.$$

*Worked number:* a model with per-config 95% CI of ±5% has $s\approx5\%/1.96\approx2.55\%$, so it can only resolve configs more than $2.8\times2.55\%\approx\mathbf{7\%}$ apart — a 5%-apart pair is a coin-flip. This is the precise sense in which "15% off but ranks correctly" (systematic, cancels) beats "5% off but mis-orders" (random, floors the resolution at 7%): the two errors act on *different* axes, and only the random one limits DSE.

**Composed error adds in quadrature.** Chain $k$ stages with independent relative errors $\epsilon_i$ (SD $\sigma_i$). To first order $\prod_i(1+\epsilon_i)\approx1+\sum_i\epsilon_i$, so the variances add and the composed relative error is

$$\sigma_{\text{tot}}=\sqrt{\textstyle\sum_i\sigma_i^2}.$$

*Worked number:* perf ±10%, power ±20%, thermal ±10% compose to $\sqrt{0.10^2+0.20^2+0.10^2}=\sqrt{0.06}\approx\mathbf{\pm24.5\%}$ — dominated by the largest term (the ±20% power model), the quantitative reason to sharpen the *worst* stage first. But quadrature assumes *independence*; if the errors are correlated (same sign — a shared technology-knob bias), they add **linearly** to $0.10+0.20+0.10=\pm40\%$. The gap between 24.5% and 40% is exactly why the perf→power→thermal loop must be **co-validated** rather than validated stage-by-stage: independent-looking errors that are secretly correlated forfeit the quadrature cancellation.

**What calibration can and cannot remove.** Calibration fits free parameters (latencies, penalties, McPAT knobs) to measured points, driving out *parameter* error — but it cannot remove *structural* error. A model with no shared-resource event has no $\rho$ (§3, §7), and **no** setting of its parameters will make it show bandwidth saturation; a model with no wrong-path (§4) cannot be tuned into wrong-path pollution. So the calibrated ~20–30% floor is the **structural residual** — the error of the model's *form*, not its constants — and the only way beneath it is more structure (a finer rung), not more fitting. This is why "calibrated ≈ 20–30%, uncalibrated ≈ 2×" is a floor and not a slope.

The mature stance: quote simulated numbers with their provenance and error class, use the model for *relative* comparisons whenever possible (trends survive calibration error better than absolutes), and reserve absolute claims for validated, calibrated configurations.

---

## 9. Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Slowdown identity | $S=wf/R_h\approx w$ (work/sim-cycle) | the whole ladder in one accounting line (§2.1) |
| Functional-only slowdown | 1–10× (JIT); ~10²–10³× (interp.) | correctness/ICount is cheap; interp. is the ~MIPS end (§2.1) |
| Event-driven cycle-approximate slowdown | 10³–10⁴× | the working rung for µarch studies |
| Cycle-accurate slowdown | 10⁴–10⁵× (10⁶ w/ detailed mem; ~1 day per sim-second) | why sampling is mandatory (§2.1) |
| RTL simulation slowdown | 10⁶–10⁷× (~1 yr per sim-second) | why you don't DSE on RTL (§2.1) |
| Event- vs cycle-driven cost | $O(E\log\bar n)$ vs $O(C{\cdot}M)$ | cost tracks *activity*, not *capacity* (§3) |
| Time-ordered invariant | pop-min = temporal order iff latencies $\ge0$ | the DES correctness backbone (§3) |
| Trace-driven wrong-path overhead | $\phi_{\text{wp}}=b(1-a)W_{\text{wp}}$ | trace-error $\propto(1-a)$; core vs memory study (§4) |
| Sampling standard error | $\text{SE}=\sigma/\sqrt n$; $n\ge(z c_v/\varepsilon)^2$ | CLT bound → computable CI (§5.1) |
| ±2% CPI @ 95% conf. | $n\approx10^4$ intervals ($\approx$ ~1% coverage) | why SimPoint/SMARTS work (§5.1) |
| Warm-up (cold-start) bias | floors RMSE at $\lvert\beta_{\text{bias}}\rvert$; $\sqrt n$ can't fix | the silent systematic error (§5.1) |
| Compute cost model | $(L,II,P)$ per op-class — minimal sufficient | latency + throughput + width (§6) |
| Achieved bandwidth | output $=\min(\text{demand},\,B\cdot\text{burst}/t_{RC})$ | BW is a result of contention, not an input (§7) |
| Calibrated architectural perf/power error | ~20–30% (structural floor) | the DSE-rung error bar; fitting can't beat it (§8) |
| Cycle-accurate (validated-to-target) error | <5% | the sign-off-ish rung |
| DSE ranking resolution | $\Delta>2.8\,s$ (random-error SD $s$) | systematic bias cancels, random error floors it (§8) |
| Composed error | $\sqrt{\sum_i\sigma_i^2}$ (indep.); linear if correlated | co-validate the perf→power→thermal loop (§8) |
| Queueing latency law | $\sim 1/(1-\rho)$ | why latency explodes near saturation |
| CPI (interval model) | base $+\sum(\text{miss rate}\times\text{penalty})$ | closed-form core performance |
| Summary statistic | geomean for ratios, arithmetic for rates | the reporting-correctness rule |

---

## Cross-references

- **Down the stack:** [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (analytical kernels + the NeuSim worked example), [OoO_Execution](../02_CPU/03_OoO_Execution.md) (the structures an O3 timing model tracks), [Memory](../03_Memory/03_Memory.md) / [DDR_Controller](../03_Memory/04_DDR_Controller.md) (what DRAM simulators encode).
- **Up the stack:** [Full_Chip_Modeling](../01_Modeling/02_Full_Chip_Modeling.md) (composing leaf models into a chip; the full tool/fidelity ladder and the perf→power→thermal loop).
- **Sibling pages (this folder):** per-tool deep dives — [gem5](02_gem5.md) (the event engine + O3 cost model of §3/§6 made concrete), [DRAM_Simulators](03_DRAM_Simulators.md) (the banks/scheduler realizing §7's bandwidth-as-output), [GPU_Simulators](04_GPU_Simulators.md), [Accelerator_and_NPU_Simulators](05_Accelerator_and_NPU_Simulators.md), [Other_Architecture_Simulators](06_Other_Architecture_Simulators.md), and [Analytical_Models](07_Analytical_Models.md) (the closed-form dual of §6's interval model). Folder [index](00_Index.md).
