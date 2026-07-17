# Xiangshan (香山) — Reading an Open OoO Core as a Sequence of Design Decisions

> **Prerequisites:** [OoO_Execution](03_OoO_Execution.md) (the window structures and their sizing knees), [Branch_Prediction_Deep_Dive](04_Branch_Prediction_Deep_Dive.md) (TAGE, the FTQ, the mispredict tax), [Cache_Microarchitecture](../03_Memory/01_Cache_Microarchitecture.md) (AMAT, MSHR/MLP, inclusion), [RISC_V_ISA](02_RISC_V_ISA.md).
> **Hands off to:** [ACE_and_CHI](../04_Interconnect/02_ACE_and_CHI.md) (the coherence fabric Kunminghu moves to), [AHB_AXI_APB](../04_Interconnect/01_AHB_AXI_APB.md) (TileLink/SoC integration), [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the design-space walk each generation performs).

---

## 0. Why this page exists

The [OoO](03_OoO_Execution.md), [Branch](04_Branch_Prediction_Deep_Dive.md), and [Cache](../03_Memory/01_Cache_Microarchitecture.md) pages give you the *models* — Little's law for the ROB, the wakeup–select recurrence for the scheduler, the $W\times P$ tax for the front end, the MSHR/MLP ceiling for the memory system. What they cannot give you is the sight of a real team **resolving those models into numbers** under a fixed target and a real area/timing budget, because that resolution normally happens inside proprietary RTL no one outside the design house ever reads.

Xiangshan (香山) removes that wall. It is a **fully open-source, server-class RV64 out-of-order core** written in Chisel at ICT/UCAS and BOSC, targeting ARM Cortex-A76/A78-class performance. Because the RTL, the parameters, and four generations of history are all public, Xiangshan is the one core where you can watch every trade-off on the sibling pages get *decided* — and, uniquely, **re-decided three times** as the team climbs toward its target.

So this page is not a block-diagram tour. It reads Xiangshan as a **sequence of design decisions** — pipeline depth, fetch/decode width, the decoupled front end, the branch predictor, the ROB/IQ/PRF window, the load-store unit, the memory hierarchy — asking of each: *what did they pick, which theory curve does that point sit on, why there and not elsewhere, and how did the pick move from Nanhu to Kunminghu?* The generational deltas are the most valuable thing an open core offers: a controlled experiment in **what to spend the next transistor budget on**, and the answer is never "make it wider."

---

## 1. The design as a set of constraints

A microarchitecture is not chosen in a vacuum; it is the fixed point of a few hard constraints. Xiangshan's four constraints predict almost every number that follows:

- **A performance target — A76/A78 class.** This fixes the *IPC* the window must deliver (~3), which via Little's law fixes the ROB, PRF, IQ, and LSQ sizes. It is a mainstream-mobile target, not a hyperscale-server one, which is why the window lands far below Golden Cove's.
- **A modest frequency target on a maturing node — 1.0–1.5 GHz on TSMC 28 nm, 2 GHz+ on advanced nodes.** This fixes the *pipeline depth* and, through the wakeup–select recurrence, caps how large the scheduler may be. Frequency, not IPC, turns out to be Xiangshan's whole gap to A76 (§1.1).
- **Open + agile, in Chisel.** Every window size is a Scala parameter, so the design can be re-instantiated cheaply. This is *why the numbers move between generations* at all — the cost of trying a bigger ROB is a recompile plus a [DiffTest](https://github.com/OpenXiangShan/difftest) co-simulation run against the NEMU reference model, not a multi-year respin.
- **Correctness first, then performance.** A first generation that boots Linux and passes co-simulation is worth more than a fast one that does not, which is why the earliest generation deliberately leaves performance on the table (§8).

### 1.1 Where the gap to A76 actually lives — and why it dictates the whole strategy

Performance is the product of two independent factors, and confusing them is the classic mistake:

$$
\text{Perf} \;=\; \text{IPC}\times f_{clk}, \qquad
\frac{\text{Perf}_{\text{XS}}}{\text{Perf}_{\text{A76}}} \;=\; \underbrace{\frac{\text{IPC}_{\text{XS}}}{\text{IPC}_{\text{A76}}}}_{\text{microarchitecture}}\times\underbrace{\frac{f_{\text{XS}}}{f_{\text{A76}}}}_{\text{mostly process node}}
$$

Plug in Nanhu (IPC $\approx 2.8$, 1.2 GHz on 28 nm) against A76 (IPC $\approx 3.5$, 2.8 GHz on 7 nm):

$$
\frac{2.8}{3.5}\times\frac{1.2}{2.8} \;=\; 0.80 \times 0.43 \;\approx\; 0.34
$$

The lesson is in the *split*, not the product. Nanhu's **IPC is already ~80 % of A76** — the microarchitecture is competitive. The gap is almost entirely the **0.43 frequency factor**, which is a 28 nm-vs-7 nm process story, not an architecture story. This single decomposition explains the entire evolution to come: since the IPC deficit is small, later generations rationally spend their budget on **frequency** (advanced nodes, and a scheduler kept small enough to clock high, §5.3) and on the *last few points* of IPC (memory-level parallelism and branch accuracy, §5–§7), and **not** on more width — which would attack the factor that is already near parity.

**Quantifying the lever — the split *is* the roadmap.** Because $\text{Perf}$ is a *product*, the marginal return on improving one ratio is set by the level of the *other* ($\partial\text{Perf}/\partial r_f=r_{\text{IPC}}$ and conversely, where $r_f,r_{\text{IPC}}$ are the frequency and IPC ratios to A76), so budget flows to whichever ratio is lower — here $r_f=0.43$, far below $r_{\text{IPC}}=0.80$. Move one factor at a time off the Nanhu point (product $0.34$):

- *All-in on width/IPC*: push $r_{\text{IPC}}$ from $0.80$ to a heroic $0.95$ at fixed clock → product $0.34\to0.41$, **+19 %**, every point bought with quadratic scheduler/PRF area (§2, §5.3).
- *All-in on the node*: hold $r_{\text{IPC}}=0.80$ and lift $r_f$ from $0.43$ (28 nm) to $0.71$ (an advanced-node $\sim\!2$ GHz vs A76's 2.8 GHz) → product $0.34\to0.80\times0.71=\mathbf{0.57}$, **+65 %, with no microarchitecture change at all.**

A 65 %-vs-19 % split of return is not close, and it decides four generations of spending. This is the iron law ([Performance_Modeling_and_DSE §2.1](../01_Modeling/01_Performance_Modeling_and_DSE.md)) used as a compass: optimize the product, and the product points at the smaller factor — frequency.

### 1.2 The generations we will track

| Codename | Gen | ROB | Interconnect | The bet this generation makes |
|---|---|---|---|---|
| **Yanqihu** 雁栖湖 | 1 (2020) | ~128 | TileLink | Get a 6-wide OoO pipeline correct and co-simulating. |
| **Nanhu** 南湖 | 2 (2022) | **192** | TileLink | Buy IPC: TAGE-SC front end, deeper window, refined LSU. |
| **Kunminghu** 昆明湖 | 3 (2023) | **256** | **CHI** | Buy MLP and scale: bigger window, vector, coherent server fabric. |
| **Kunminghu v2** | 3.5 (2024) | 256 | CHI-B/C | Buy the *last* MLP + accuracy: wider IQ/LSQ, 6-table TAGE, stash coherence. |

Each is a clean iteration — same ISA (RV64GCB, +V from Kunminghu), same 6-wide shape, but the window, predictor, and memory system move. Unless noted, figures below are **Nanhu with Kunminghu deltas called out**, because the deltas *are* the lesson.

### 1.3 The agile loop, quantified — why the numbers can move at all

The first three constraints name a target; the fourth — *open + agile, in Chisel, correctness-first* — is what makes hitting it **affordable**, and it deserves to be stated as a productivity argument rather than a virtue. The design-space walk this whole page narrates (§8) is governed by the DSE budget inequality ([Performance_Modeling_and_DSE §5](../01_Modeling/01_Performance_Modeling_and_DSE.md)):

$$
\underbrace{|\text{configs tried}|}_{\text{coverage}}\times\underbrace{c_{\text{eval}}}_{\text{cost per config}} \;\le\; \text{budget},
$$

where $c_{\text{eval}}$ = the wall-clock (and engineer) cost of resolving one design point to a *trustworthy* number. Agile methodology is a direct attack on $c_{\text{eval}}$ on two fronts.

**Chisel collapses the *instantiate* cost.** Because every window size is a Scala parameter, changing the ROB from 192 to 256 is a generate-and-recompile, not a hand re-pipelining of RTL — the cost of *trying* a point drops from a partial redesign (person-months) to a build (hours). This is exactly what turns §5's sizing formulas from paper exercises into things a real team *sweeps*: ROB, IQ, LQ/SQ, MSHR, and TAGE-table counts are all generator parameters, so the diminishing-returns knees of the OoO and Cache pages get found by measurement, not asserted a priori.

**DiffTest collapses the *verify* cost — by a log factor.** [DiffTest](https://github.com/OpenXiangShan/difftest) runs the core in lock-step against the NEMU reference model, comparing architectural state at every committed instruction. Its value is *bug localization*. A divergence that first shows a symptom at dynamic instruction $M$ (a hung boot after $M\sim10^{9}$ instructions) costs, under end-of-run checking, an $O(\log M)$ bisection over the trace to isolate — $\sim\!\log_2 10^{9}\approx 30$ re-runs. Instruction-lockstep checking flags the *exact* diverging instruction the cycle after it retires, cutting those $\sim\!30$ debug iterations to $\mathbf{1}$:

$$
\text{debug iterations to localize a divergence} \;:\quad O(\log M)\ \longrightarrow\ O(1).
$$

That $\log M$ is why "correctness first" is not a tax but an *enabler*: a co-simulated core that boots Linux is reached cheaply, so the first generation (Yanqihu) can spend its budget on being *correct* without forfeiting years — and every later generation inherits a harness that makes each swept config trustworthy at near-zero marginal cost.

**The payoff is the cadence.** Collapsing $c_{\text{eval}}$ lets coverage rise inside a fixed budget, and the visible result is a **~annual tape-out cadence** (Yanqihu 2020 → Nanhu 2021–22 → Kunminghu 2023 → v2 2024) where a from-scratch commercial core re-tunes its window *once* per 3–5-year program. Xiangshan re-decides the §5 sizes **three times** in that window (192→256 ROB, 4→6 TAGE tables, 16→24 MSHRs) because the agile loop made re-deciding cheap. Read §8's trajectory as this inequality running on silicon: cheap evaluations bought the coverage.

---

## 2. Pipeline depth and width: the first two bets

Depth and width are the two most expensive, least reversible decisions a core makes, and Xiangshan's choices only make sense against its constraints.

**Depth — why Xiangshan is short (~11 stages) when commercial cores are 15–19.** Pipeline depth trades frequency against the branch penalty ([CPU_Architecture §1.3](01_CPU_Architecture.md)): more stages shrink the per-stage logic so $f_{clk}$ rises, but they lengthen the fetch-to-resolve distance, which *is* the mispredict penalty $P$. The [$W\times P$ tax](04_Branch_Prediction_Deep_Dive.md) makes that penalty compound with width:

$$
\frac{\text{IPC}_{eff}}{W} \;=\; \frac{1}{1 + W\cdot\dfrac{\text{MPKI}}{1000}\cdot P}
$$

where $W$ = issue width, $P$ = mispredict penalty in cycles, MPKI = mispredicts per 1000 instructions. A 5 GHz core *must* run 15–19 stages to make timing, and then pays a $P\approx 14$–17 penalty it can only afford because it has driven $m$ (per-branch mispredict rate) down for years. Xiangshan targets only 1–2 GHz, so a **short ~11-stage pipe already makes timing** — and a short pipe buys a small $P\approx 6$–8, which is doubly valuable early: it keeps the tax bounded *even while the predictor is still maturing* (Nanhu's $m$ is not yet best-in-class). A first-gen open core is right to stay shallow; frequency is nearly free at a modest target, and a short pipe forgives an immature predictor and an immature verification flow.

**Width — why 6-wide, and why it never grows.** Width trades IPC against the quadratic port cost of the PRF and scheduler ([OoO §2.3](03_OoO_Execution.md), [§4.3](03_OoO_Execution.md)). Xiangshan picks **6-wide decode/rename/dispatch/commit**, matching its A76-class target. It does *not* go wider, and the reason is the **fetch-width wall** ([Branch §7.2](04_Branch_Prediction_Deep_Dive.md)): the first taken branch truncates a fetch block, so useful instructions per fetch are bounded by

$$
\mathbb{E}[\text{useful insns per fetch}] \;\approx\; \frac{1}{b\cdot t} \;\approx\; \frac{1}{0.2\times 0.6}\;\approx\; 8
$$

where $b$ = branch density, $t$ = taken fraction. On integer code a machine wider than ~8 buys almost nothing unless it can predict past multiple taken branches per cycle. So 6-wide sits just below that knee, and **width is the one knob that stays frozen across all four generations** — because it is past the point of return, and every transistor spent widening would be better spent on the frequency factor (§1.1) that is actually behind.

**Width, quantified — why lanes 7–8 are dead weight, and why it attacks the wrong factor.** Put a number on "buys almost nothing." Model the position of the first taken branch as geometric with per-instruction taken probability $p=b\,t=0.2\times0.6=0.12$ ([Branch §7.2](04_Branch_Prediction_Deep_Dive.md)); a decoder of width $w$ then delivers $\mathbb{E}[\min(w,L)]=\sum_{k=1}^{w}(1-p)^{k-1}=\dfrac{1-(1-p)^{w}}{p}$ useful instructions before the block truncates, where $L$ = run length to the first taken branch. So

$$
\text{lanes 1–6 deliver } \frac{1-0.88^{6}}{0.12}=\frac{0.536}{0.12}\approx 4.47,\qquad
\text{lanes 7–8 add } (1-p)^6+(1-p)^7=0.88^6+0.88^7\approx 0.87,
$$

i.e. going $6\to8$-wide lifts effective decode only $4.47\to5.34$ — **+19 %** — because on integer code a taken branch has already truncated the block more than half the time before lane 7 ever fires ($0.88^6\approx0.46$). Now price the other side: the wakeup–select CAM cost is $\propto S\,W^2$ ([OoO §4.3](03_OoO_Execution.md)), so $6\to8$ multiplies scheduler compare-and-broadcast work by $(8/6)^2=1.78$ — **+78 %** — *and* lengthens the wires inside the single-cycle recurrence, forcing a clock give-back. Fold both into the iron law $\text{Perf}=\text{IPC}\times f$: a +19 % IPC that costs even a 12 % frequency give-back nets $1.19\times0.88=1.05$ — a rounding error — while spending 78 % more scheduler area *and* attacking the frequency factor §1.1 just showed is the *entire* deficit. Width past 6 is not merely low-return here; it is **negative**. That is why the knob is welded shut for four generations while every other number moves.

The teaching point: the two irreversible macro-decisions were made *once*, correctly, against the target — and the entire subsequent evolution happens in the *sizes* of structures, never in depth or width. That is what a mature design-space walk looks like ([Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md)): freeze the expensive axes early, iterate on the cheap ones.

---

## 3. The decoupled front end: the FTQ as the front-end analogue of the ROB

Fetch must produce a next-PC every cycle, but the predictor's best answer takes two cycles to compute and the I-cache stalls on its own misses. Couple them and each stalls the other. The fix is a queue between them — the **Fetch Target Queue (FTQ)** — and the concept is exactly the OoO window's, moved to the front ([Branch §7.1](04_Branch_Prediction_Deep_Dive.md)): **decouple a producer (prediction) from a consumer (fetch) with a buffer so the producer can run ahead.** Xiangshan's BPU sprints ahead into a ~48-entry FTQ; two payoffs follow, and they are why every modern front end is decoupled:

1. **Latency hiding.** The multi-cycle TAGE lookup leaves the per-cycle fetch critical path; the FTQ absorbs the bubbles.
2. **Prefetch for free.** The run-ahead addresses sitting in the FTQ *are* an I-cache prefetch stream, so target lines arrive before fetch reaches them — decisive on the large-footprint, front-end-bound code a server-class core must run.

**What an FTQ entry must remember — derived, not dumped.** An FTQ entry is not a signal list; it holds exactly what its two jobs demand. Job one, *drive fetch*: the block's start PC and fall-through address. Job two — the subtle one — *train and recover the predictor when this block finally resolves*: the entry therefore carries the prediction metadata (which TAGE table provided, the RAS top-of-stack pointer). Read that metadata not as fields but as **the predictor's undo-and-train log**: when the branch resolves correct it updates the providers named here; when it resolves wrong, the redirect and the RAS restoration both read from here. Everything in an FTQ entry is one of those two jobs.

**Fetch is wider than decode, and that is deliberate.** Xiangshan fetches up to **8 instructions (32 B, a half cache line) per cycle** but decodes only 6. Fetching wider than you decode looks wasteful until you recall the fetch-width wall: taken branches truncate blocks *below* the nominal width, so over-fetching keeps the 6-wide decoder fed on the cycles a block runs short. The 8-vs-6 asymmetry is the §2 fetch-width math made concrete in silicon.

**How deep is ~48, derived — Little's law twice over.** The FTQ depth is not arbitrary; it is fixed by the two jobs an entry does, each a Little's-law occupancy ($N=\lambda W$, the same identity that sizes the ROB — [Branch §7.1](04_Branch_Prediction_Deep_Dive.md)). An FTQ entry is born at prediction and freed only when its block's branches have all resolved/committed (it holds the undo-and-train metadata until then), so its residency is a full predict-to-commit lifetime, and two demands stack:

1. **Cover the in-flight window.** Every fetch block with an instruction still in the pipe needs a live FTQ entry. Feeding an $N_{ROB}=192$ window at $\sim\!6.5$ useful instructions per block (the truncated run of §2) makes the resident block count $N_{\text{blk}}\approx N_{ROB}/6.5\approx 30$ — Little's law with the block's predict-to-commit lifetime as $W$.
2. **Run ahead to prefetch.** On top of that, the BPU must sprint far enough in front of fetch that a target line is *requested* before fetch arrives: $D_{\text{ahead}}\gtrsim L_{\text{miss}}/t_{\text{cyc}}$ blocks. Hiding an L2 hit ($\sim\!13$ cyc) at $\sim\!1$ block/cyc is $\sim\!13$–18 blocks; a deeper L3-ward miss wants more.

Add them: $30+\sim\!18\approx\mathbf{48}$. The two terms are the FTQ's two payoffs made quantitative — the ~30 is the window-coverage that lets prediction *not stall* the back end, the ~18 is the runahead that makes the FTQ a free I-prefetcher. A server-class, front-end-bound footprint (§7) pushes the second term up, which is exactly why decoupled front ends run *deep* queues rather than the handful of entries a latency-hiding buffer alone would need.

---

## 4. Branch prediction: an accuracy machine that buys down the tax

The front end exists to make the mandatory guess wrong as rarely as possible, because on a 6-wide core the mispredict tax dominates and its only two knobs are *cut $P$* (done, via the short pipe of §2) or *cut $m$* ([Branch §0.1](04_Branch_Prediction_Deep_Dive.md)). Xiangshan's entire BPU is an $m$-reduction machine, and it is the clearest place on the page to watch a team pay for accuracy by the half-percent.

**Why a hybrid of specialized predictors, not one big table.** The next-PC needs several different facts answered early, and each fact has a different *nature*, so each needs its own structure ([Branch §1](04_Branch_Prediction_Deep_Dive.md)). Xiangshan composes the standard set:

- **TAGE-SC** for conditional direction — the theoretical centerpiece. Keeping several tagged tables at *geometrically* spaced history lengths is the minimum-table covering of a correlation need that lives on a log scale, and the tags let very long histories be trusted without paying the aliasing tax those histories would otherwise impose ([Branch §4](04_Branch_Prediction_Deep_Dive.md)). The statistical corrector (SC) overrides TAGE on low-confidence cases for a further ~0.5 %.
- **ITTAGE** for indirect targets — the same geometric-history idea applied to predicting an *address* rather than a bit ([Branch §5.1](04_Branch_Prediction_Deep_Dive.md)).
- **RAS** for returns, because a return's target is a property of **call context**, not of the branch PC, so it cannot be a PC-indexed table at all ([Branch §6](04_Branch_Prediction_Deep_Dive.md)).

This is the same TAGE-SC-L + ITTAGE family that ships in Intel P-cores and SiFive's P870 — Xiangshan is simply the one you can read end to end.

**The evolution is the lesson: half a percent is worth a generation's effort.** Nanhu runs ~4 tagged TAGE tables at ~97 % accuracy; Kunminghu v2 grows to **6 tables and ~97.5 %**. That sounds trivial until you price it through the tax. Accuracy enters $m$ linearly, and on a 6-wide core the tax multiplies by $W$; a drop from 3.0 % to ~2.5 % mispredicts, at $P\approx 7$, is worth more realized IPC than most of the window growth elsewhere on this page — which is *why* geometric reach (grows in the table *count*, [Branch §4.2](04_Branch_Prediction_Deep_Dive.md)) is the lever the team keeps pulling, despite each new table costing storage and lookup power. It echoes the [OoO worked problem](03_OoO_Execution.md) that "prediction, not width, is the first lever," and here you can see a real team spend accordingly.

**The accuracy the pipe demands — a worked target.** Why is 97 %→97.5 % worth a generation? Because the *required* accuracy is fixed by the machine's own depth×width, and Xiangshan can compute its target. From the [Branch §0.2](04_Branch_Prediction_Deep_Dive.md) floor, holding the branch tax to a fraction $\tau$ of ideal CPI needs

$$
p_{\text{miss}}^\star \;\le\; \frac{\tau}{f_{\text{br}}\,W\,P}, \qquad f_{\text{br}}=0.2,\ W=6,\ P\approx7\ (\text{the short pipe of §2}),
$$

where $p_{\text{miss}}^\star$ = tolerable per-branch mispredict rate and $\tau$ = allowed tax-to-ideal ratio. For $\tau=0.1$ (surrender $\sim\!9$ % of peak): $p_{\text{miss}}^\star\le 0.1/(0.2\cdot6\cdot7)=0.0119$ → **≥98.8 % accuracy** to be "on budget." Nanhu's 97 % ($m=0.03$) *misses* that bar — its realized fraction is $1/(1+W f_{\text{br}} m P)=1/(1+6\cdot0.2\cdot0.03\cdot7)=1/1.252=80$ %, i.e. a fifth of the machine surrendered — which is precisely the pressure driving the predictor investment. **And the short pipe is what makes 97 % survivable at all:** run the *same* 97 % predictor on a 5 GHz-class 17-stage pipe ($P\approx17$) and the tax balloons to $6\cdot0.2\cdot0.03\cdot17=0.61$, realized $1/1.61=62$ % — a 38-point loss. Xiangshan's $P\approx7$ roughly *halves* the accuracy pressure of a deep core, which is why a first-gen open core can ship an immature predictor and still land at 80 % of peak (§2's "a short pipe forgives an immature predictor," now a number).

**Why the table *count* is the lever — the $\rho^2$ tax, checked against silicon.** Cutting the mispredict rate costs storage quadratically: $S\propto p_{\text{miss}}^{-2}$ ([Branch §0.2](04_Branch_Prediction_Deep_Dive.md)). Nanhu → Kunminghu-v2 moves accuracy $97.0\%\to97.5\%$, i.e. $m:0.030\to0.025$, a reduction factor $\rho=0.030/0.025=1.2$, so theory predicts $\rho^2=1.44\times$ predictor storage. The team grew the tagged tables **4→6**, i.e. $1.5\times$ — within rounding of the $1.44\times$ the model demands. The generational table-count bump is not stylistic; it is the $\rho^2$ price of the last half-percent, spent exactly where the theory says it must be, and because geometric reach grows in table *count* each new table both lengthens history *and* pays that $\rho^2$ — which is why "add a table" is the move that keeps recurring.

---

## 5. The out-of-order window: three structures, three different theory curves

The window — ROB, PRF, issue queues, and (in §6) the LSQ — is one coordinated bet on how far ahead the machine may run. Its aggregate size is a bandwidth–delay product ([OoO §3.2](03_OoO_Execution.md)): to sustain IPC while each instruction lives $\bar{T}_{res}$ cycles in flight, the window must hold $N \gtrsim \text{IPC}\times\bar{T}_{res}$. But the three structures sit on *different* curves and are provisioned against *different* demands — the single most instructive thing about reading real numbers.

### 5.1 ROB: 192 → 256, sized between the miss shadow and the branch horizon

The ROB is the program-order ledger, and its size is squeezed between one force pushing up and two pushing down ([OoO §3.2](03_OoO_Execution.md)):

$$
\underbrace{\text{IPC}\times\frac{L_{miss}}{\text{MLP}}}_{\text{miss shadow (push up)}} \;\lesssim\; N_{ROB} \;\lesssim\; \underbrace{\frac{1000}{\text{MPKI}}}_{\text{branch horizon (push down)}}
$$

where $L_{miss}$ = miss latency, MLP = independent misses overlapped, MPKI = mispredicts per 1000 instructions. The **branch horizon** is the ceiling: past it, the far end of the ROB holds instructions that will on average be squashed before they commit. Nanhu's ~3 % per-branch mispredict rate gives $\text{MPKI}=1000\,b\,m=1000\times0.2\times0.03=6$ (branch density $b\approx0.2$, mispredict rate $m\approx0.03$), so $N_{useful}\approx 1000/6 \approx 167$ — and **192 sits right at that horizon**, exactly where theory says growth stops paying for this predictor. The **miss shadow** sets the floor: with an A76-class memory system (L2 at 10–15 cycles, not a 100 ns server DRAM window), $L_{miss}$ and the useful MLP are modest, so the floor sits below the horizon and the two meet in the ~200 range — which is precisely why Xiangshan lands at 192–256 and **not** at Golden Cove's 512. That 512 is the *server* answer to the same equation (huge $L_{miss}$, high MLP demand); Xiangshan's target simply puts the knee somewhere else.

**The floor, worked — and why it is not 600.** Take the miss-shadow floor literally. To hide a *single isolated* DRAM miss ($L_{miss}\approx200$ cyc) at the target $\text{IPC}\approx3$ needs, by Little's law with $\text{MLP}=1$, $N_{ROB}\gtrsim3\times200=600$ entries — unbuildable, and more than $3\times$ Nanhu's 192 (the [OoO §3.2](03_OoO_Execution.md) verdict: *you cannot outlast an isolated DRAM miss by window depth*). Xiangshan does not try. Its floor is the *MLP-amortized* shadow: overlap $\text{MLP}\approx4$ independent misses and the requirement collapses to $N_{ROB}\gtrsim\text{IPC}\times L_{miss}/\text{MLP}=3\times200/4=\mathbf{150}$. With the branch horizon at $\sim\!167$ (above), the floor ($\sim\!150$) sits just under it, and the two bracket the **192** the core ships. The reading is exact: 192 is about the smallest window that both hides a 4-way-overlapped DRAM shadow *and* stays inside the speculation horizon this predictor sustains. (Illustrative $\text{MLP}$ and $L_{miss}$; the *structure* — floor just below horizon — is what pins the ~200 landing.)

The **Nanhu→Kunminghu bump, 192→256**, is then legible: it is the team spending window on **MLP** as the improved predictor (§4) pushed the branch horizon out far enough to justify a deeper speculative window. You buy ROB depth *after* you buy predictor accuracy, never before — the horizon has to move first.

**Why the horizon must move first — the diminishing return, computed.** The mean run-length $1/q$ is only the ceiling; the *useful occupancy* of an $N$-deep ROB is the geometric sum $E[\text{useful}]=(1-(1-q)^{N})/q$ with $q=\text{MPKI}/1000$ (the survival-of-younger-entries derivation, [OoO §3.2](03_OoO_Execution.md)). Evaluate it at both predictors:

| Predictor | $q$ | $E[\text{useful}]$ @ $N{=}192$ | @ $N{=}256$ | 192→256 return |
|---|---|---|---|---|
| Nanhu (97 %, MPKI 6) | 0.006 | 114 | 131 | **+15 %** |
| Improved (97.5 %, MPKI 5) | 0.005 | 124 | 145 | **+17 %** |

The marginal entry at $N{=}256$ commits with probability $e^{-qN}$: at the old $q=0.006$ that is $e^{-1.54}=0.21$, at the improved $q=0.005$ it is $e^{-1.28}=0.28$ — the predictor upgrade **raises the value of the 256th slot by a third** *before a single ROB entry is added*. That is the quantified form of "buy accuracy first": deepening 192→256 under the old predictor climbs toward a 167-entry ceiling and returns +15 %; doing it *after* the predictor pushes the ceiling to 200 returns +17 % against a horizon that no longer caps it. The far entries are worthless until the horizon moves — so it moves first.

### 5.2 PRF: 128 INT / 96 FP — the under-provisioning gamble, taken aggressively

The physical register file has a hard floor: to guarantee rename never stalls for a tag, $N_{phys} \ge N_{arch} + N_{ROB}$ ([OoO §2.3](03_OoO_Execution.md)). For Nanhu that floor is $32 + 192 = 224$. **Xiangshan ships 128** — far below the floor. This is not a bug; it is the standard under-provisioning bet ([OoO worked problem 3](03_OoO_Execution.md)) taken hard, and the reconciliation is a genuinely deep point about *why the ROB and PRF are sized against different demands*:

- The **ROB is provisioned for the tail** — the rare, deep miss shadow where 192 instructions really are in flight — because a too-small ROB *caps the MLP window* and throws away the whole point of speculation.
- The **PRF is provisioned for the body** — steady-state occupancy, which is far below 192, scaled by the register-write fraction $f\approx 0.6$–0.7 (stores, branches, and compares write no register). A too-small PRF costs only an occasional *dispatch bubble* when the free list drains — recoverable, not catastrophic.

So 128 is matched to typical occupancy $\times f$, while 192 is matched to the peak. Xiangshan takes this bet **aggressively** (128 is below even the expected-demand estimate $32 + f\cdot N_{ROB}$ for a full 192-ROB) because the PRF is a heavily multiported RAM whose area *and* access time grow quadratically with port count, and that access time sits on the load-use critical path — the exact thing a 28 nm core chasing frequency cannot afford to lengthen ([OoO §2.3](03_OoO_Execution.md)). The core consciously accepts occasional rename stalls to keep the PRF small and fast. Recovery uses **RAT checkpoints** snapshotted at each branch (one-cycle restore, [OoO §2.5](03_OoO_Execution.md)) rather than a slow ROB-walk — the high-performance choice, paid for with ~32–48 checkpoints for in-flight branches.

**How often does the gamble actually cost? — the free-list drain, worked.** Quantify the "occasional" stall. With $N_{phys}=128$ INT and $N_{arch}=32$, there are $128-32=96$ tags free to hand to speculative writers. Rename stalls only when in-flight register-writers reach 96, i.e. when ROB occupancy $O$ satisfies $f\cdot O\ge 96$ at write fraction $f\approx0.65$ — that is $O\ge 96/0.65\approx\mathbf{148}$, or **77 % of the 192-entry ROB full of register-writers**. On steady-state code the occupancy sits far below 148 (dispatch and commit roughly balance), so the free list rarely drains and the bet pays. The elegant part is *when* it doesn't: the ROB only climbs toward 148–192 in a **deep miss shadow**, exactly when the head is stalled on a DRAM miss and the machine is already memory-bound — so the rename bubble the small PRF induces lands *inside* a stall the core is eating anyway, at near-zero marginal cost. That is why Xiangshan can under-provision the PRF *below* its $\sim\!157$-entry expected demand ($32+0.65\times192$): the stalls it causes are hidden under the very miss shadows that filled the window. The PRF is sized for the common-case *body*; the rare tail where it binds is already latency-bound on memory. (Reported 128/96; the robust claim is the inequality $N_{phys}<N_{arch}+f\,N_{ROB}$ — under-provisioned by construction.)

### 5.3 Issue queues: distributed and small, because the scheduler sets the clock

The scheduler is the one structure whose latency *is* a lower bound on the cycle time: the wakeup→select→issue loop for back-to-back dependent instructions is a **single-cycle recurrence that cannot be pipelined away**, and its cost grows roughly as $N\times S\times N_{cdb}$ — the "$N^2$" scheduler wall ([OoO §4.3](03_OoO_Execution.md)). For a frequency-sensitive core this dictates the two defining choices:

- **Distributed, not unified.** Xiangshan splits the scheduler into small per-class queues — **Int 32, FP 16, Mem 16, AGU 16**. This is the Zen-4 side of the [OoO §4.5](03_OoO_Execution.md) trade: smaller $N$ per queue and physically shorter wakeup wires stretch the single-cycle loop less, buying frequency and power, at the cost of *fragmentation* (a burst of integer-ready ops cannot borrow an idle FP slot). For a core whose whole deficit is frequency (§1.1), the fragmentation loss is small and the timing win is exactly what it needs — the opposite of Golden Cove's unified 128-entry scheduler, which takes the utilization win and pays it back in wakeup power.
- **Small, and tag-only.** The queues hold *tags, not values* — readiness, not data — so entries stay a handful of bits and the queue can be both fast and only as deep as the recurrence allows.

The **Kunminghu v2 delta — Int IQ 32→40, FP 16→24** — is the team cautiously enlarging the scheduling window once the advanced-node timing headroom made a slightly larger $N$ affordable, buying a wider view of ready instructions without breaking the loop. Note it grows the queues by ~25 %, not 2×: the recurrence punishes large $N$ hard, so even a confident team steps this knob gently.

**Why the step is gentle — the recurrence delay, quantified.** The wakeup→select loop must close in one cycle, and its delay rises with queue size roughly as the CAM match-line plus tag-broadcast wire, $\sim\!\sqrt{N}$ ([OoO §4.3](03_OoO_Execution.md); the same $\sqrt{\cdot}$ array law the Cache page applies to SRAM capacity). Xiangshan's *modest* frequency target helps: a 2 GHz cycle is $\sim\!500$ ps, $\sim\!30$ FO4 at an advanced-node $\sim\!17$ ps FO4, versus a 5 GHz core's $\sim\!13$ FO4 — real headroom to spend on a bigger scheduler. But the loop is unpipelinable, so the headroom is finite and the growth must be priced. Int IQ $32\to40$ ($\times1.25$ in $N$) lengthens the recurrence delay by $\sqrt{40/32}=\sqrt{1.25}=1.118$ — **+12 % wakeup delay** — which fits inside the slack a new process node returns, and no more. A $2\times$ queue would be $\sqrt2=1.41$, a +41 % hit no node upgrade absorbs — hence 25 %, not 2×. The distributed split (Int/FP/Mem/AGU) is the other half of the same budget: four queues of $\sim\!N/4$ each keep every $\sqrt{N}$ short, buying frequency at the fragmentation cost §5.3 already judged acceptable for a frequency-bound target.

---

## 6. The load-store unit: the late-binding name space in practice

Registers can be renamed because their names are known at decode; memory cannot, because whether a load aliases an older store depends on *addresses* that are not known until execute. Memory is a second, **late-binding name space**, and the LSQ is the engine that *discovers* its dependences dynamically ([OoO §5](03_OoO_Execution.md)). Xiangshan's LSU is the worked example of every choice on that page.

**Speculate by default, predict the exceptions.** A load with unresolved older stores ahead of it can wait (safe, slow) or assume no alias and issue now (fast, but a genuine alias costs a ~10–15-cycle replay flush). Because violations are rare and stalls are frequent, the expected cost favors speculation, and a **store-set predictor** ([OoO §5.1](03_OoO_Execution.md)) closes the gap by stalling *only* the specific loads that history says alias — driving the per-load cost toward $\min(\bar{C}_{stall}, p_{viol}C_{flush})$. Xiangshan implements exactly this, plus store-to-load forwarding from the SQ and commit-time store writes to the D-cache ([OoO §5.2–5.3](03_OoO_Execution.md)), so a speculative store leaves no architectural trace until it retires.

**Why LQ 64 / SQ 48, and why they scale with the window.** Memory ops are ~35–40 % of instructions, so the memory queues must cover the in-flight memory ops the ROB exposes, with headroom for clustering:

$$
N_{LQ}+N_{SQ} \;\approx\; f_{mem}\times N_{ROB}\times(\text{clustering headroom})
$$

Nanhu's $64+48 = 112$ against a 192-ROB is a ratio of ~0.58 — comfortably above the ~0.4 memory fraction, so memory ops rarely block dispatch even when they bunch. The revealing fact is that **Kunminghu v2 grows LQ 64→80 and SQ 48→64 while the 256-ROB holds** — preserving that ~0.56 ratio. The whole window scales *coherently*; you do not deepen the ROB without widening the memory queues that feed it, or the narrowest stage silently caps the MLP you paid for.

**Per-type, via the same Little's law.** Split the window fraction by op type ([OoO §5](03_OoO_Execution.md)): with load fraction $f_{ld}\approx0.22$ and store fraction $f_{st}\approx0.13$, the bare in-flight demand against a 192-ROB is $N_{LQ}\gtrsim f_{ld}N_{ROB}\approx0.22\times192\approx42$ loads and $N_{SQ}\gtrsim f_{st}N_{ROB}\approx0.13\times192\approx25$ stores. Nanhu ships **64 / 48** — $1.5\times$ and $1.9\times$ those floors. The headroom is not slack: loads and stores *cluster* (a copy loop is nearly all memory ops, transiently $f_{mem}\!\to\!1$), and the SQ carries the extra duty of holding *committed* stores through their cache-drain latency, so its residency $\bar T_{res}$ exceeds a load's and Little's law returns a richer count — the same reason Zen 4 runs 64 SQ against a bare $\sim\!48$ ([OoO §5](03_OoO_Execution.md)). The v2 growth to **80 / 64** simply re-applies the formula at $N_{ROB}=256$: $0.22\times256\approx56$ and $0.13\times256\approx33$, times the same clustering headroom.

**The clearest "buy MLP" move on the page.** Kunminghu v2 grows LQ, SQ, **and** L2 MSHRs 16→24 *together*. That is not three tweaks; it is one decision, because memory-level parallelism is a chain — LQ slot → SQ slot → D-cache MSHR → L2 MSHR — and the MSHR count *is* the level's MLP ceiling ([Cache §3.2](../03_Memory/01_Cache_Microarchitecture.md)). On a memory-bound SPEC target, overlapping more independent misses is where the last IPC lives, so the team widens the *entire* chain in lockstep. Widening any one stage alone would have bought nothing.

**The MSHR count is a Little's law, worked.** Why 16→24 L2 MSHRs, specifically? The MSHR file is a queue of outstanding misses, so $N_{MSHR}\ge\lambda_{miss}\times L_{miss}$ ([Cache §3.2](../03_Memory/01_Cache_Microarchitecture.md)) — and read the other way, $N_{MSHR}$ *is* the level's MLP ceiling, the most independent misses whose latencies overlap. To cut an $L_{miss}\approx200$-cyc DRAM penalty (from L2's vantage) to an exposed $\sim\!12$ cyc, the demanded overlap is $\text{MLP}=200/12\approx17$, so an L2 tracking **24** has just enough ceiling to reach it with margin, while **16** caps exposure at $200/16=12.5$ cyc — but only if the window can *supply* the misses. It can: a 256-ROB at $f_{mem}\approx0.35$ holds $\sim\!90$ in-flight memory ops, whose independent-miss subset on memory-bound SPEC is comfortably in the teens. So 16→24 is the MSHR ceiling lifted to match the MLP the *enlarged* 256-window now exposes — and it is why LQ ($64\to80$), SQ ($48\to64$), ROB ($192\to256$), and L2 MSHR ($16\to24$) all grow $\sim\!1.3\times$ together: MLP is a chain, and a chain is only as wide as its narrowest link.

---

## 7. The memory hierarchy and interconnect: from private cache to coherent fabric

The cache hierarchy exists to make the ROB's miss shadow shallow enough that the window can hide it ([Cache §6](../03_Memory/01_Cache_Microarchitecture.md)) — every level is another chance to catch a miss before it reaches the ~200-cycle DRAM latency that drives §5.1's floor. Xiangshan's L1s are conventional (**64 KB, 8-way, 64 B lines**, 2-cycle I / 3-cycle D), and their one non-obvious constraint is worth keeping because it is a real design lesson: with 8 ways and 64 B lines the index spans

$$
\text{index+offset} \;=\; \log_2\!\frac{64\,\text{KB}}{8\times 64\,\text{B}} + \log_2 64 \;=\; 7 + 6 \;=\; 13\text{ bits},
$$

one bit past the 12-bit page offset. That single overhang is why a **VIPT** cache this size must either resolve the aliasing bit or accept the tag-after-TLB check — the concrete reason L1 capacity, associativity, and page size are not independent knobs.

**L2 is inclusive — a deliberate coherence choice.** Xiangshan's 1 MB (Nanhu) → 1–2 MB (Kunminghu) L2 is **inclusive** of the L1s and directory-tracked (MESI). Inclusion duplicates L1 capacity inside L2, which looks wasteful, but it buys a cheap coherence filter: the L2 directory alone can answer whether any L1 holds a line, so external probes never disturb the core ([Cache §6.2](../03_Memory/01_Cache_Microarchitecture.md)). For a core built to scale to multicore, paying capacity for probe-filtering is the right side of that curve.

**The biggest architectural decision on the page: Nanhu TileLink → Kunminghu CHI.** This is Xiangshan declaring server intent, and it is a textbook instance of the coherence trade-off ([ACE_and_CHI §9](../04_Interconnect/02_ACE_and_CHI.md)). TileLink is a simple, open, tree-friendly protocol — perfect for a few cores below the broadcast knee. CHI is a **directory/mesh** protocol built for the $O(N^2)$ snoop wall that broadcast hits past ~8–16 cores ([ACE_and_CHI §3](../04_Interconnect/02_ACE_and_CHI.md)). Moving to CHI is the team choosing the point on that curve set by *many* cores, accepting directory storage and home-hop latency in exchange for coherence traffic that scales as $O(N\bar{K})$ instead of $O(N^2)$. Kunminghu v2's **CHI Issue B/C** then adds an enhanced snoop filter and **stash** hints (a core proactively pushes shared-dirty lines toward the home node) — the multicore analogue of prefetching, cutting *coherence* latency the way a prefetcher cuts *miss* latency.

**Prefetching: the other way to shrink the miss shadow.** Prefetching converts misses to hits before they stall ([Cache §7](../03_Memory/01_Cache_Microarchitecture.md)) — the same MLP goal as a bigger ROB, attacked from the memory side, which makes window growth and prefetching **substitutes for one objective**. Xiangshan buys both, and escalates the prefetcher each generation: Nanhu stream+stride → Kunminghu adds **BOP** (Best-Offset Prefetching, which learns the offset that maximizes L2 hit rate) → v2 adds a per-page stride detector. A team that already spent on the window still spends on prefetch, because they hit the same MLP ceiling from two directions.

**What the hierarchy buys — a target-AMAT number.** Compose the levels with the recursive AMAT ([Cache §1.2](../03_Memory/01_Cache_Microarchitecture.md)), $\text{AMAT}=t_{L1}+m_{L1}(t_{L2}+m_{L2}(t_{L3}+m_{L3}t_{DRAM}))$, using Xiangshan's latencies and *illustrative* SPEC-class local miss rates ($t_{L1}{=}3,\,m_{L1}{=}0.04$; $t_{L2}{=}13,\,m_{L2}{=}0.35$; $t_{L3}{=}35,\,m_{L3}{=}0.4$; $t_{DRAM}{=}200$ cyc):

$$
\text{AMAT}=3+0.04\big(13+0.35\,(35+0.4\times200)\big)=3+0.04\,(13+0.35\times115)=3+0.04\times53.25\approx\mathbf{5.1}\ \text{cyc.}
$$

The hierarchy has defanged a 200-cycle DRAM into a ~5-cycle average — and the reason the L1 is **64 KB, not 32 KB** falls out of the same equation: on the reuse-distance curve ([Cache §1.3](../03_Memory/01_Cache_Microarchitecture.md)) doubling L1 capacity drops $m_{L1}$ (say $0.06\to0.04$), and because $m_{L1}$ multiplies *every* term below it, that one factor cuts the miss half of AMAT by a third — worth more than any downstream tweak. Now close the loop with §5.1: the *global* rate reaching DRAM is $m_{L1}m_{L2}m_{L3}=0.04\times0.35\times0.4=0.0056$, so the residual DRAM exposure is $0.0056\times200\approx1.1$ cyc per access — and it is exactly this residual, spread over the memory-op stream, that the 192–256 ROB and the 16→24 MSHRs (§6) exist to overlap. Cache sizing and window sizing are one problem: the hierarchy sets how deep the miss shadow is, and the window is built to hide precisely what the hierarchy leaves. (Miss rates illustrative; the ~5-cycle AMAT and the sub-1 % global DRAM rate are the structural facts.)

---

## 8. Reading the evolution as a trade-off trajectory

The payoff of an open core is that you can lay its generations side by side and see *which curve each one climbed* — and the pattern is a clean diminishing-returns walk.

| Generation | The knob that moved | The theory curve it was climbing |
|---|---|---|
| **Yanqihu** | Get 6-wide OoO correct + co-simulating | none — correctness, the precondition for optimization |
| **Nanhu** | TAGE-SC front end; ROB 128→192; refined LSU | branch tax ($m$, [Branch §0.1](04_Branch_Prediction_Deep_Dive.md)) + window ([OoO §3.2](03_OoO_Execution.md)) |
| **Kunminghu** | ROB 192→256; TileLink→CHI; +vector; +BOP | MLP/window ([OoO §3.2](03_OoO_Execution.md)) + coherence scaling ([ACE §3](../04_Interconnect/02_ACE_and_CHI.md)) |
| **Kunminghu v2** | IQ/LQ/SQ/MSHR up; 4→6 TAGE tables; CHI-B/C | the *last* MLP ([Cache §3.2](../03_Memory/01_Cache_Microarchitecture.md)) + accuracy ([Branch §4](04_Branch_Prediction_Deep_Dive.md)) |

Two meta-lessons fall out, and they are the reason to study the trajectory rather than any single snapshot:

1. **Width and depth never move; every generation spends on MLP and prediction.** Those are the levers with slope left — the miss shadow and the branch horizon still have room, while width is past the fetch-width knee (§2) and depth is fixed by the frequency target. This is the concrete shape of a design-space walk: freeze the axes that are past their knee, and pour the budget into the ones that still return ([Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md)).
2. **The strategy is exactly what §1.1's decomposition predicted.** Because the IPC gap to A76 was already small (~0.8) and the gap was *frequency*, the team chases frequency through process nodes and a scheduler kept deliberately small to clock high (§5.3), and closes the residual IPC through MLP and accuracy — never through the width that would attack the factor already near parity. Reading the numbers, you are watching a rational response to where the gap actually lives.

3. **Every spend is *coherent* and *cheap to try* — the two enablers.** The Kunminghu-v2 numbers move *together* by nearly one factor — ROB $192\to256$ ($1.33\times$), SQ $48\to64$ ($1.33\times$), LQ $64\to80$ ($1.25\times$), L2 MSHR $16\to24$ ($1.5\times$), TAGE $4\to6$ tables ($1.5\times$) — because MLP and accuracy are *chains* (§6, §4) whose weakest link caps the rest, so a generation lifts the whole chain or wastes the effort. And it can afford to, three times over, only because §1.3's agile loop drove the cost of *trying* a chain-width down to a recompile-plus-DiffTest. The trajectory is the DSE budget inequality ([Performance_Modeling_and_DSE §5](../01_Modeling/01_Performance_Modeling_and_DSE.md)) running on real silicon: cheap, trustworthy evaluations bought the coverage to find these knees by measurement rather than argument.

That is the whole value of an open core: not the block diagram, but the chance to see a real team resolve the sibling pages' models into numbers, discover the diminishing returns, and re-spend accordingly — three times, in public.

---

## 9. Numbers to memorize

| Parameter | Nanhu | Kunminghu (Δ) | Why this value (section) |
|---|---|---|---|
| Decode / rename / commit width | 6 | 6 | A76-class target, below the fetch-width wall (§2) |
| Fetch width | 8 insns (32 B) | 8 | over-fetch past taken-branch truncation (§3) |
| Pipeline depth | ~11 stages | ~11 | short pipe → small $P$, forgives immature predictor (§2) |
| ROB entries | **192** | **256** | between miss shadow and branch horizon (§5.1) |
| Physical regs (INT / FP) | **128 / 96** | 128 / 96 | under-provisioned below the $N_{arch}+N_{ROB}$ floor (§5.2) |
| Checkpoints (RAT) | 32–48 | 48–64 | one-cycle branch recovery, in-flight branch count (§5.2) |
| Issue queues (Int/FP/Mem/AGU) | 32 / 16 / 16 / 16 | v2: 40 / 24 / … | wakeup–select recurrence, distributed for frequency (§5.3) |
| Load queue / Store queue | 64 / 48 | v2: 80 / 64 | memory fraction of the window, scaled coherently (§6) |
| L1 I / D cache | 64 KB 8-way, 2 / 3 cyc | same | VIPT sizing constraint (§7) |
| L1 D / L2 MSHRs | 16 / 16 | v2 L2: 24 | the MLP ceiling of each level (§6) |
| L2 | 1 MB, 8-way, inclusive, 10–15 cyc | 1–2 MB | probe-filtering via inclusion (§7) |
| Branch predictor | TAGE-SC + ITTAGE + RAS | 4→6 TAGE tables | geometric-history accuracy machine (§4) |
| Branch accuracy / mispredict penalty | ~97 % / 6–8 cyc | ~97.5 % | cuts the $W\times P$ tax (§2, §4) |
| Interconnect | TileLink | **CHI / CHI-B/C** | directory scaling past the snoop wall (§7) |
| Frequency | 1.0–1.5 GHz (28 nm) | 1.8–2.0+ GHz (adv. node) | the real gap to A76 (§1.1) |
| SPEC CPU2006 IPC | 2.5–3.0 (~80 % of A76) | 3.5+ (target) | window + prediction + MLP (§5–§7) |

**Derived design-point checks (the theory each real number sits on):**

| Quantity | Value | Derivation (section) |
|---|---|---|
| Perf split to A76: IPC vs frequency lever | +19 % (all-IPC) vs +65 % (all-node) | iron-law compass (§1.1) |
| DiffTest debug localization | $O(\log M)\to O(1)$ iterations | lock-step co-sim (§1.3) |
| Width $6\to8$: IPC gain vs wakeup cost | +19 % IPC vs +78 % CAM ($W^2$) | fetch-wall + $O(W^2)$ (§2) |
| FTQ depth ≈ window-blocks + runahead | $\sim\!30+\sim\!18\approx48$ | Little's law ×2 (§3) |
| Required accuracy $p_{\text{miss}}^\star=\tau/(f_{\text{br}}WP)$ | ≥98.8 % @ $W{=}6,P{=}7,\tau{=}0.1$ | branch-tax floor (§4) |
| Realized peak @ 97 %: short vs deep pipe | 80 % ($P{=}7$) vs 62 % ($P{=}17$) | short pipe forgives predictor (§4) |
| Predictor storage tax, 97→97.5 % | $\rho^2=1.44\times$ ≈ the 4→6 tables ($1.5\times$) | $S\propto p_{\text{miss}}^{-2}$ (§4) |
| ROB floor: isolated-miss vs MLP-4 | 600 (unbuildable) vs 150 | miss shadow (§5.1) |
| $E[\text{useful}]$ of 192-ROB @ MPKI 6 / 5 | 114 / 124 (256-ROB: 131 / 145) | branch-horizon geometric sum (§5.1) |
| PRF free-list drain threshold | ROB $\ge$ 77 % full of writers | under-provision gamble (§5.2) |
| IQ $32\to40$ recurrence-delay cost | $\sqrt{1.25}=+12\%$ (vs $2\times$: +41 %) | wakeup $\sqrt N$ (§5.3) |
| Full-hierarchy AMAT | ≈ 5.1 cyc (against 200-cyc DRAM) | recursive AMAT (§7) |
| Coherent window scaling (v2) | ROB/LQ/SQ/MSHR/TAGE all $\sim\!1.3\times$ | MLP/accuracy chain (§6, §8) |

**Memory latencies that set the knees:** L1 3–4 cyc · L2 10–15 · DRAM 100–300 — the $L_{miss}$ that drives the §5.1 ROB floor and the §6 MSHR count.

---

## 10. Cross-references

- **Down the stack (the models this page instantiates):** [OoO_Execution](03_OoO_Execution.md) (the ROB/IQ/PRF/LSQ derivations and sizing knees §5–§6 make concrete), [Branch_Prediction_Deep_Dive](04_Branch_Prediction_Deep_Dive.md) (the TAGE, FTQ, and $W\times P$-tax theory behind §2–§4), [Cache_Microarchitecture](../03_Memory/01_Cache_Microarchitecture.md) (AMAT, MSHR/MLP, and inclusion behind §6–§7).
- **The exact derivation each § instantiates:** the [OoO §3.2](03_OoO_Execution.md) Little's-law ROB and geometric branch-horizon (§5.1), the [OoO §2.3](03_OoO_Execution.md) $N_{arch}+N_{ROB}$ PRF floor (§5.2), the [OoO §4.3](03_OoO_Execution.md) $S\,W^2$ wakeup recurrence (§2, §5.3), the [OoO §5](03_OoO_Execution.md) LQ/SQ window fractions (§6), the [Branch §0.2](04_Branch_Prediction_Deep_Dive.md) $p_{\text{miss}}^\star\propto1/(WP)$ accuracy floor and $S\propto p_{\text{miss}}^{-2}$ storage tax (§4), the [Branch §7.1–7.2](04_Branch_Prediction_Deep_Dive.md) FTQ Little's law and fetch-width wall (§2–§3), the [Cache §1.2](../03_Memory/01_Cache_Microarchitecture.md) recursive AMAT and [§3.2](../03_Memory/01_Cache_Microarchitecture.md) MSHR Little's law (§6–§7), and the [Performance_Modeling_and_DSE §5](../01_Modeling/01_Performance_Modeling_and_DSE.md) DSE budget inequality the agile trajectory runs on (§1.3, §8).
- **Up / adjacent (what this core plugs into):** [ACE_and_CHI](../04_Interconnect/02_ACE_and_CHI.md) (the coherence trade-off behind the TileLink→CHI move, §7), [AHB_AXI_APB](../04_Interconnect/01_AHB_AXI_APB.md) (TileLink and SoC integration), [TLB_and_Virtual_Memory](../03_Memory/02_TLB_and_Virtual_Memory.md) (the iTLB/dTLB in the fetch and load paths), [Performance_Modeling_and_DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) (the design-space walk §8 reads as a trajectory).
- **Prerequisite:** [RISC_V_ISA](02_RISC_V_ISA.md) (the RV64GCB(V) namespace being renamed and the trap model at commit), [CPU_Architecture](01_CPU_Architecture.md) (the pipeline-depth/frequency trade of §2).

---

## 11. References

1. X. Ge et al., "Towards Developing High Performance RISC-V Processors Using Agile Methodology," *MICRO 2022*. The Xiangshan methodology paper (all three artifact-evaluation badges).
2. **OpenXiangShan/XiangShan** repository and design docs: [github.com/OpenXiangShan/XiangShan](https://github.com/OpenXiangShan/XiangShan), [docs.xiangshan.cc](https://docs.xiangshan.cc/projects/design/).
3. A. Seznec and P. Michaud, "A case for (partially) TAgged GEometric history length branch prediction," *JILP*, 2006; A. Seznec, "TAGE-SC-L Branch Predictors," *JILP-CBP*, 2014. The direction predictor of §4.
4. A. Seznec, "A 64-Kbytes ITTAGE Indirect Branch Predictor," *JILP*, 2011. The indirect predictor of §4.
5. G. Chrysos and J. Emer, "Memory Dependence Prediction Using Store Sets," *ISCA*, 1998. The store-set policy of §6.
6. P. Michaud, "Best-Offset Hardware Prefetching," *HPCA 2016*. The BOP prefetcher of §7.
7. Arm Ltd., *AMBA CHI Architecture Specification* (Issue B/C). The coherence fabric of §7.
8. J. L. Hennessy and D. A. Patterson, *Computer Architecture: A Quantitative Approach*, 6th ed., Morgan Kaufmann, 2017. Ch. 3 for the window and ILP-limit models §5 builds on.

---

## Navigation

| Direction | Link |
|---|---|
| **Previous** | [Branch Prediction Deep Dive](04_Branch_Prediction_Deep_Dive.md) |
| **Next** | [ACE and CHI](../04_Interconnect/02_ACE_and_CHI.md) |
| **Up** | [CPU Architecture](01_CPU_Architecture.md) |
| **Related** | [OoO Execution](03_OoO_Execution.md) · [Cache Microarchitecture](../03_Memory/01_Cache_Microarchitecture.md) · [Performance Modeling and DSE](../01_Modeling/01_Performance_Modeling_and_DSE.md) |
