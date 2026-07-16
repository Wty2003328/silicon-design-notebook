# Xiangshan (香山) — Reading an Open OoO Core as a Sequence of Design Decisions

> **Prerequisites:** [OoO_Execution](05_OoO_Execution.md) (the window structures and their sizing knees), [Branch_Prediction_Deep_Dive](06_Branch_Prediction_Deep_Dive.md) (TAGE, the FTQ, the mispredict tax), [Cache_Microarchitecture](07_Cache_Microarchitecture.md) (AMAT, MSHR/MLP, inclusion), [RISC_V_ISA](04_RISC_V_ISA.md).
> **Hands off to:** [ACE_and_CHI](12_ACE_and_CHI.md) (the coherence fabric Kunminghu moves to), [AHB_AXI_APB](11_AHB_AXI_APB.md) (TileLink/SoC integration), [Performance_Modeling_and_DSE](01_Performance_Modeling_and_DSE.md) (the design-space walk each generation performs).

---

## 0. Why this page exists

The [OoO](05_OoO_Execution.md), [Branch](06_Branch_Prediction_Deep_Dive.md), and [Cache](07_Cache_Microarchitecture.md) pages give you the *models* — Little's law for the ROB, the wakeup–select recurrence for the scheduler, the $W\times P$ tax for the front end, the MSHR/MLP ceiling for the memory system. What they cannot give you is the sight of a real team **resolving those models into numbers** under a fixed target and a real area/timing budget, because that resolution normally happens inside proprietary RTL no one outside the design house ever reads.

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

### 1.2 The generations we will track

| Codename | Gen | ROB | Interconnect | The bet this generation makes |
|---|---|---|---|---|
| **Yanqihu** 雁栖湖 | 1 (2020) | ~128 | TileLink | Get a 6-wide OoO pipeline correct and co-simulating. |
| **Nanhu** 南湖 | 2 (2022) | **192** | TileLink | Buy IPC: TAGE-SC front end, deeper window, refined LSU. |
| **Kunminghu** 昆明湖 | 3 (2023) | **256** | **CHI** | Buy MLP and scale: bigger window, vector, coherent server fabric. |
| **Kunminghu v2** | 3.5 (2024) | 256 | CHI-B/C | Buy the *last* MLP + accuracy: wider IQ/LSQ, 6-table TAGE, stash coherence. |

Each is a clean iteration — same ISA (RV64GCB, +V from Kunminghu), same 6-wide shape, but the window, predictor, and memory system move. Unless noted, figures below are **Nanhu with Kunminghu deltas called out**, because the deltas *are* the lesson.

---

## 2. Pipeline depth and width: the first two bets

Depth and width are the two most expensive, least reversible decisions a core makes, and Xiangshan's choices only make sense against its constraints.

**Depth — why Xiangshan is short (~11 stages) when commercial cores are 15–19.** Pipeline depth trades frequency against the branch penalty ([CPU_Architecture §1.3](03_CPU_Architecture.md)): more stages shrink the per-stage logic so $f_{clk}$ rises, but they lengthen the fetch-to-resolve distance, which *is* the mispredict penalty $P$. The [$W\times P$ tax](06_Branch_Prediction_Deep_Dive.md) makes that penalty compound with width:

$$
\frac{\text{IPC}_{eff}}{W} \;=\; \frac{1}{1 + W\cdot\dfrac{\text{MPKI}}{1000}\cdot P}
$$

where $W$ = issue width, $P$ = mispredict penalty in cycles, MPKI = mispredicts per 1000 instructions. A 5 GHz core *must* run 15–19 stages to make timing, and then pays a $P\approx 14$–17 penalty it can only afford because it has driven $m$ (per-branch mispredict rate) down for years. Xiangshan targets only 1–2 GHz, so a **short ~11-stage pipe already makes timing** — and a short pipe buys a small $P\approx 6$–8, which is doubly valuable early: it keeps the tax bounded *even while the predictor is still maturing* (Nanhu's $m$ is not yet best-in-class). A first-gen open core is right to stay shallow; frequency is nearly free at a modest target, and a short pipe forgives an immature predictor and an immature verification flow.

**Width — why 6-wide, and why it never grows.** Width trades IPC against the quadratic port cost of the PRF and scheduler ([OoO §2.3](05_OoO_Execution.md), [§4.3](05_OoO_Execution.md)). Xiangshan picks **6-wide decode/rename/dispatch/commit**, matching its A76-class target. It does *not* go wider, and the reason is the **fetch-width wall** ([Branch §7.2](06_Branch_Prediction_Deep_Dive.md)): the first taken branch truncates a fetch block, so useful instructions per fetch are bounded by

$$
\mathbb{E}[\text{useful insns per fetch}] \;\approx\; \frac{1}{b\cdot t} \;\approx\; \frac{1}{0.2\times 0.6}\;\approx\; 8
$$

where $b$ = branch density, $t$ = taken fraction. On integer code a machine wider than ~8 buys almost nothing unless it can predict past multiple taken branches per cycle. So 6-wide sits just below that knee, and **width is the one knob that stays frozen across all four generations** — because it is past the point of return, and every transistor spent widening would be better spent on the frequency factor (§1.1) that is actually behind.

The teaching point: the two irreversible macro-decisions were made *once*, correctly, against the target — and the entire subsequent evolution happens in the *sizes* of structures, never in depth or width. That is what a mature design-space walk looks like ([Performance_Modeling_and_DSE](01_Performance_Modeling_and_DSE.md)): freeze the expensive axes early, iterate on the cheap ones.

---

## 3. The decoupled front end: the FTQ as the front-end analogue of the ROB

Fetch must produce a next-PC every cycle, but the predictor's best answer takes two cycles to compute and the I-cache stalls on its own misses. Couple them and each stalls the other. The fix is a queue between them — the **Fetch Target Queue (FTQ)** — and the concept is exactly the OoO window's, moved to the front ([Branch §7.1](06_Branch_Prediction_Deep_Dive.md)): **decouple a producer (prediction) from a consumer (fetch) with a buffer so the producer can run ahead.** Xiangshan's BPU sprints ahead into a ~48-entry FTQ; two payoffs follow, and they are why every modern front end is decoupled:

1. **Latency hiding.** The multi-cycle TAGE lookup leaves the per-cycle fetch critical path; the FTQ absorbs the bubbles.
2. **Prefetch for free.** The run-ahead addresses sitting in the FTQ *are* an I-cache prefetch stream, so target lines arrive before fetch reaches them — decisive on the large-footprint, front-end-bound code a server-class core must run.

**What an FTQ entry must remember — derived, not dumped.** An FTQ entry is not a signal list; it holds exactly what its two jobs demand. Job one, *drive fetch*: the block's start PC and fall-through address. Job two — the subtle one — *train and recover the predictor when this block finally resolves*: the entry therefore carries the prediction metadata (which TAGE table provided, the RAS top-of-stack pointer). Read that metadata not as fields but as **the predictor's undo-and-train log**: when the branch resolves correct it updates the providers named here; when it resolves wrong, the redirect and the RAS restoration both read from here. Everything in an FTQ entry is one of those two jobs.

**Fetch is wider than decode, and that is deliberate.** Xiangshan fetches up to **8 instructions (32 B, a half cache line) per cycle** but decodes only 6. Fetching wider than you decode looks wasteful until you recall the fetch-width wall: taken branches truncate blocks *below* the nominal width, so over-fetching keeps the 6-wide decoder fed on the cycles a block runs short. The 8-vs-6 asymmetry is the §2 fetch-width math made concrete in silicon.

---

## 4. Branch prediction: an accuracy machine that buys down the tax

The front end exists to make the mandatory guess wrong as rarely as possible, because on a 6-wide core the mispredict tax dominates and its only two knobs are *cut $P$* (done, via the short pipe of §2) or *cut $m$* ([Branch §0.1](06_Branch_Prediction_Deep_Dive.md)). Xiangshan's entire BPU is an $m$-reduction machine, and it is the clearest place on the page to watch a team pay for accuracy by the half-percent.

**Why a hybrid of specialized predictors, not one big table.** The next-PC needs several different facts answered early, and each fact has a different *nature*, so each needs its own structure ([Branch §1](06_Branch_Prediction_Deep_Dive.md)). Xiangshan composes the standard set:

- **TAGE-SC** for conditional direction — the theoretical centerpiece. Keeping several tagged tables at *geometrically* spaced history lengths is the minimum-table covering of a correlation need that lives on a log scale, and the tags let very long histories be trusted without paying the aliasing tax those histories would otherwise impose ([Branch §4](06_Branch_Prediction_Deep_Dive.md)). The statistical corrector (SC) overrides TAGE on low-confidence cases for a further ~0.5 %.
- **ITTAGE** for indirect targets — the same geometric-history idea applied to predicting an *address* rather than a bit ([Branch §5.1](06_Branch_Prediction_Deep_Dive.md)).
- **RAS** for returns, because a return's target is a property of **call context**, not of the branch PC, so it cannot be a PC-indexed table at all ([Branch §6](06_Branch_Prediction_Deep_Dive.md)).

This is the same TAGE-SC-L + ITTAGE family that ships in Intel P-cores and SiFive's P870 — Xiangshan is simply the one you can read end to end.

**The evolution is the lesson: half a percent is worth a generation's effort.** Nanhu runs ~4 tagged TAGE tables at ~97 % accuracy; Kunminghu v2 grows to **6 tables and ~97.5 %**. That sounds trivial until you price it through the tax. Accuracy enters $m$ linearly, and on a 6-wide core the tax multiplies by $W$; a drop from 3.0 % to ~2.5 % mispredicts, at $P\approx 7$, is worth more realized IPC than most of the window growth elsewhere on this page — which is *why* geometric reach (grows in the table *count*, [Branch §4.2](06_Branch_Prediction_Deep_Dive.md)) is the lever the team keeps pulling, despite each new table costing storage and lookup power. It echoes the [OoO worked problem](05_OoO_Execution.md) that "prediction, not width, is the first lever," and here you can see a real team spend accordingly.

---

## 5. The out-of-order window: three structures, three different theory curves

The window — ROB, PRF, issue queues, and (in §6) the LSQ — is one coordinated bet on how far ahead the machine may run. Its aggregate size is a bandwidth–delay product ([OoO §3.2](05_OoO_Execution.md)): to sustain IPC while each instruction lives $\bar{T}_{res}$ cycles in flight, the window must hold $N \gtrsim \text{IPC}\times\bar{T}_{res}$. But the three structures sit on *different* curves and are provisioned against *different* demands — the single most instructive thing about reading real numbers.

### 5.1 ROB: 192 → 256, sized between the miss shadow and the branch horizon

The ROB is the program-order ledger, and its size is squeezed between one force pushing up and two pushing down ([OoO §3.2](05_OoO_Execution.md)):

$$
\underbrace{\text{IPC}\times\frac{L_{miss}}{\text{MLP}}}_{\text{miss shadow (push up)}} \;\lesssim\; N_{ROB} \;\lesssim\; \underbrace{\frac{1000}{\text{MPKI}}}_{\text{branch horizon (push down)}}
$$

where $L_{miss}$ = miss latency, MLP = independent misses overlapped, MPKI = mispredicts per 1000 instructions. The **branch horizon** is the ceiling: past it, the far end of the ROB holds instructions that will on average be squashed before they commit. Nanhu's ~3 % mispredict rate gives MPKI $\approx 0.2\times 30 \approx 6$, so $N_{useful}\approx 1000/6 \approx 170$ — and **192 sits right at that horizon**, exactly where theory says growth stops paying for this predictor. The **miss shadow** sets the floor: with an A76-class memory system (L2 at 10–15 cycles, not a 100 ns server DRAM window), $L_{miss}$ and the useful MLP are modest, so the floor sits below the horizon and the two meet in the ~200 range — which is precisely why Xiangshan lands at 192–256 and **not** at Golden Cove's 512. That 512 is the *server* answer to the same equation (huge $L_{miss}$, high MLP demand); Xiangshan's target simply puts the knee somewhere else.

The **Nanhu→Kunminghu bump, 192→256**, is then legible: it is the team spending window on **MLP** as the improved predictor (§4) pushed the branch horizon out far enough to justify a deeper speculative window. You buy ROB depth *after* you buy predictor accuracy, never before — the horizon has to move first.

### 5.2 PRF: 128 INT / 96 FP — the under-provisioning gamble, taken aggressively

The physical register file has a hard floor: to guarantee rename never stalls for a tag, $N_{phys} \ge N_{arch} + N_{ROB}$ ([OoO §2.3](05_OoO_Execution.md)). For Nanhu that floor is $32 + 192 = 224$. **Xiangshan ships 128** — far below the floor. This is not a bug; it is the standard under-provisioning bet ([OoO worked problem 3](05_OoO_Execution.md)) taken hard, and the reconciliation is a genuinely deep point about *why the ROB and PRF are sized against different demands*:

- The **ROB is provisioned for the tail** — the rare, deep miss shadow where 192 instructions really are in flight — because a too-small ROB *caps the MLP window* and throws away the whole point of speculation.
- The **PRF is provisioned for the body** — steady-state occupancy, which is far below 192, scaled by the register-write fraction $f\approx 0.6$–0.7 (stores, branches, and compares write no register). A too-small PRF costs only an occasional *dispatch bubble* when the free list drains — recoverable, not catastrophic.

So 128 is matched to typical occupancy $\times f$, while 192 is matched to the peak. Xiangshan takes this bet **aggressively** (128 is below even the expected-demand estimate $32 + f\cdot N_{ROB}$ for a full 192-ROB) because the PRF is a heavily multiported RAM whose area *and* access time grow quadratically with port count, and that access time sits on the load-use critical path — the exact thing a 28 nm core chasing frequency cannot afford to lengthen ([OoO §2.3](05_OoO_Execution.md)). The core consciously accepts occasional rename stalls to keep the PRF small and fast. Recovery uses **RAT checkpoints** snapshotted at each branch (one-cycle restore, [OoO §2.5](05_OoO_Execution.md)) rather than a slow ROB-walk — the high-performance choice, paid for with ~32–48 checkpoints for in-flight branches.

### 5.3 Issue queues: distributed and small, because the scheduler sets the clock

The scheduler is the one structure whose latency *is* a lower bound on the cycle time: the wakeup→select→issue loop for back-to-back dependent instructions is a **single-cycle recurrence that cannot be pipelined away**, and its cost grows roughly as $N\times S\times N_{cdb}$ — the "$N^2$" scheduler wall ([OoO §4.3](05_OoO_Execution.md)). For a frequency-sensitive core this dictates the two defining choices:

- **Distributed, not unified.** Xiangshan splits the scheduler into small per-class queues — **Int 32, FP 16, Mem 16, AGU 16**. This is the Zen-4 side of the [OoO §4.5](05_OoO_Execution.md) trade: smaller $N$ per queue and physically shorter wakeup wires stretch the single-cycle loop less, buying frequency and power, at the cost of *fragmentation* (a burst of integer-ready ops cannot borrow an idle FP slot). For a core whose whole deficit is frequency (§1.1), the fragmentation loss is small and the timing win is exactly what it needs — the opposite of Golden Cove's unified 128-entry scheduler, which takes the utilization win and pays it back in wakeup power.
- **Small, and tag-only.** The queues hold *tags, not values* — readiness, not data — so entries stay a handful of bits and the queue can be both fast and only as deep as the recurrence allows.

The **Kunminghu v2 delta — Int IQ 32→40, FP 16→24** — is the team cautiously enlarging the scheduling window once the advanced-node timing headroom made a slightly larger $N$ affordable, buying a wider view of ready instructions without breaking the loop. Note it grows the queues by ~25 %, not 2×: the recurrence punishes large $N$ hard, so even a confident team steps this knob gently.

---

## 6. The load-store unit: the late-binding name space in practice

Registers can be renamed because their names are known at decode; memory cannot, because whether a load aliases an older store depends on *addresses* that are not known until execute. Memory is a second, **late-binding name space**, and the LSQ is the engine that *discovers* its dependences dynamically ([OoO §5](05_OoO_Execution.md)). Xiangshan's LSU is the worked example of every choice on that page.

**Speculate by default, predict the exceptions.** A load with unresolved older stores ahead of it can wait (safe, slow) or assume no alias and issue now (fast, but a genuine alias costs a ~10–15-cycle replay flush). Because violations are rare and stalls are frequent, the expected cost favors speculation, and a **store-set predictor** ([OoO §5.1](05_OoO_Execution.md)) closes the gap by stalling *only* the specific loads that history says alias — driving the per-load cost toward $\min(\bar{C}_{stall}, p_{viol}C_{flush})$. Xiangshan implements exactly this, plus store-to-load forwarding from the SQ and commit-time store writes to the D-cache ([OoO §5.2–5.3](05_OoO_Execution.md)), so a speculative store leaves no architectural trace until it retires.

**Why LQ 64 / SQ 48, and why they scale with the window.** Memory ops are ~35–40 % of instructions, so the memory queues must cover the in-flight memory ops the ROB exposes, with headroom for clustering:

$$
N_{LQ}+N_{SQ} \;\approx\; f_{mem}\times N_{ROB}\times(\text{clustering headroom})
$$

Nanhu's $64+48 = 112$ against a 192-ROB is a ratio of ~0.58 — comfortably above the ~0.4 memory fraction, so memory ops rarely block dispatch even when they bunch. The revealing fact is that **Kunminghu v2 grows LQ 64→80 and SQ 48→64 while the 256-ROB holds** — preserving that ~0.56 ratio. The whole window scales *coherently*; you do not deepen the ROB without widening the memory queues that feed it, or the narrowest stage silently caps the MLP you paid for.

**The clearest "buy MLP" move on the page.** Kunminghu v2 grows LQ, SQ, **and** L2 MSHRs 16→24 *together*. That is not three tweaks; it is one decision, because memory-level parallelism is a chain — LQ slot → SQ slot → D-cache MSHR → L2 MSHR — and the MSHR count *is* the level's MLP ceiling ([Cache §3.2](07_Cache_Microarchitecture.md)). On a memory-bound SPEC target, overlapping more independent misses is where the last IPC lives, so the team widens the *entire* chain in lockstep. Widening any one stage alone would have bought nothing.

---

## 7. The memory hierarchy and interconnect: from private cache to coherent fabric

The cache hierarchy exists to make the ROB's miss shadow shallow enough that the window can hide it ([Cache §6](07_Cache_Microarchitecture.md)) — every level is another chance to catch a miss before it reaches the ~200-cycle DRAM latency that drives §5.1's floor. Xiangshan's L1s are conventional (**64 KB, 8-way, 64 B lines**, 2-cycle I / 3-cycle D), and their one non-obvious constraint is worth keeping because it is a real design lesson: with 8 ways and 64 B lines the index spans

$$
\text{index+offset} \;=\; \log_2\!\frac{64\,\text{KB}}{8\times 64\,\text{B}} + \log_2 64 \;=\; 7 + 6 \;=\; 13\text{ bits},
$$

one bit past the 12-bit page offset. That single overhang is why a **VIPT** cache this size must either resolve the aliasing bit or accept the tag-after-TLB check — the concrete reason L1 capacity, associativity, and page size are not independent knobs.

**L2 is inclusive — a deliberate coherence choice.** Xiangshan's 1 MB (Nanhu) → 1–2 MB (Kunminghu) L2 is **inclusive** of the L1s and directory-tracked (MESI). Inclusion duplicates L1 capacity inside L2, which looks wasteful, but it buys a cheap coherence filter: the L2 directory alone can answer whether any L1 holds a line, so external probes never disturb the core ([Cache §6.2](07_Cache_Microarchitecture.md)). For a core built to scale to multicore, paying capacity for probe-filtering is the right side of that curve.

**The biggest architectural decision on the page: Nanhu TileLink → Kunminghu CHI.** This is Xiangshan declaring server intent, and it is a textbook instance of the coherence trade-off ([ACE_and_CHI §9](12_ACE_and_CHI.md)). TileLink is a simple, open, tree-friendly protocol — perfect for a few cores below the broadcast knee. CHI is a **directory/mesh** protocol built for the $O(N^2)$ snoop wall that broadcast hits past ~8–16 cores ([ACE_and_CHI §3](12_ACE_and_CHI.md)). Moving to CHI is the team choosing the point on that curve set by *many* cores, accepting directory storage and home-hop latency in exchange for coherence traffic that scales as $O(N\bar{K})$ instead of $O(N^2)$. Kunminghu v2's **CHI Issue B/C** then adds an enhanced snoop filter and **stash** hints (a core proactively pushes shared-dirty lines toward the home node) — the multicore analogue of prefetching, cutting *coherence* latency the way a prefetcher cuts *miss* latency.

**Prefetching: the other way to shrink the miss shadow.** Prefetching converts misses to hits before they stall ([Cache §7](07_Cache_Microarchitecture.md)) — the same MLP goal as a bigger ROB, attacked from the memory side, which makes window growth and prefetching **substitutes for one objective**. Xiangshan buys both, and escalates the prefetcher each generation: Nanhu stream+stride → Kunminghu adds **BOP** (Best-Offset Prefetching, which learns the offset that maximizes L2 hit rate) → v2 adds a per-page stride detector. A team that already spent on the window still spends on prefetch, because they hit the same MLP ceiling from two directions.

---

## 8. Reading the evolution as a trade-off trajectory

The payoff of an open core is that you can lay its generations side by side and see *which curve each one climbed* — and the pattern is a clean diminishing-returns walk.

| Generation | The knob that moved | The theory curve it was climbing |
|---|---|---|
| **Yanqihu** | Get 6-wide OoO correct + co-simulating | none — correctness, the precondition for optimization |
| **Nanhu** | TAGE-SC front end; ROB 128→192; refined LSU | branch tax ($m$, [Branch §0.1](06_Branch_Prediction_Deep_Dive.md)) + window ([OoO §3.2](05_OoO_Execution.md)) |
| **Kunminghu** | ROB 192→256; TileLink→CHI; +vector; +BOP | MLP/window ([OoO §3.2](05_OoO_Execution.md)) + coherence scaling ([ACE §3](12_ACE_and_CHI.md)) |
| **Kunminghu v2** | IQ/LQ/SQ/MSHR up; 4→6 TAGE tables; CHI-B/C | the *last* MLP ([Cache §3.2](07_Cache_Microarchitecture.md)) + accuracy ([Branch §4](06_Branch_Prediction_Deep_Dive.md)) |

Two meta-lessons fall out, and they are the reason to study the trajectory rather than any single snapshot:

1. **Width and depth never move; every generation spends on MLP and prediction.** Those are the levers with slope left — the miss shadow and the branch horizon still have room, while width is past the fetch-width knee (§2) and depth is fixed by the frequency target. This is the concrete shape of a design-space walk: freeze the axes that are past their knee, and pour the budget into the ones that still return ([Performance_Modeling_and_DSE](01_Performance_Modeling_and_DSE.md)).
2. **The strategy is exactly what §1.1's decomposition predicted.** Because the IPC gap to A76 was already small (~0.8) and the gap was *frequency*, the team chases frequency through process nodes and a scheduler kept deliberately small to clock high (§5.3), and closes the residual IPC through MLP and accuracy — never through the width that would attack the factor already near parity. Reading the numbers, you are watching a rational response to where the gap actually lives.

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

**Memory latencies that set the knees:** L1 3–4 cyc · L2 10–15 · DRAM 100–300 — the $L_{miss}$ that drives the §5.1 ROB floor and the §6 MSHR count.

---

## 10. Cross-references

- **Down the stack (the models this page instantiates):** [OoO_Execution](05_OoO_Execution.md) (the ROB/IQ/PRF/LSQ derivations and sizing knees §5–§6 make concrete), [Branch_Prediction_Deep_Dive](06_Branch_Prediction_Deep_Dive.md) (the TAGE, FTQ, and $W\times P$-tax theory behind §2–§4), [Cache_Microarchitecture](07_Cache_Microarchitecture.md) (AMAT, MSHR/MLP, and inclusion behind §6–§7).
- **Up / adjacent (what this core plugs into):** [ACE_and_CHI](12_ACE_and_CHI.md) (the coherence trade-off behind the TileLink→CHI move, §7), [AHB_AXI_APB](11_AHB_AXI_APB.md) (TileLink and SoC integration), [TLB_and_Virtual_Memory](08_TLB_and_Virtual_Memory.md) (the iTLB/dTLB in the fetch and load paths), [Performance_Modeling_and_DSE](01_Performance_Modeling_and_DSE.md) (the design-space walk §8 reads as a trajectory).
- **Prerequisite:** [RISC_V_ISA](04_RISC_V_ISA.md) (the RV64GCB(V) namespace being renamed and the trap model at commit), [CPU_Architecture](03_CPU_Architecture.md) (the pipeline-depth/frequency trade of §2).

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
| **Previous** | [Branch Prediction Deep Dive](06_Branch_Prediction_Deep_Dive.md) |
| **Next** | [ACE and CHI](12_ACE_and_CHI.md) |
| **Up** | [CPU Architecture](03_CPU_Architecture.md) |
| **Related** | [OoO Execution](05_OoO_Execution.md) · [Cache Microarchitecture](07_Cache_Microarchitecture.md) · [Performance Modeling and DSE](01_Performance_Modeling_and_DSE.md) |
