# Block Activity and Power — Estimating the One Term You Can't Read Off the Schematic

> **Prerequisites:** [CMOS_Fundamentals](../00_Fundamentals/01_CMOS_Fundamentals.md) §4 (the $\tfrac12 CV^2$ dissipated per transition, the three powers), [Power_Fundamentals](01_Power_Fundamentals.md) (the total-power equation and the system budget these estimates feed).
> **Hands off to:** [Power_Analysis_and_Signoff](05_Power_Analysis_and_Signoff.md) (SAIF/VCD annotation mechanics, IR-drop signoff), [Power_Reduction_Techniques](03_Power_Reduction_Techniques.md) (what to *do* about a high-activity net), [Full_Chip_Modeling](../01_Architecture_and_PPA/01_Modeling/02_Full_Chip_Modeling.md) (composing block power into a chip).

---

## 0. Why this page exists

Dynamic power is $P_{dyn} = \alpha\, C\, V^2 f$, and three of its four factors are already sitting in the design database. The capacitance $C$ comes from the netlist plus parasitic extraction; the voltage $V$ and frequency $f$ come from the operating point you chose. The fourth factor — $\alpha$, how often each node actually toggles — is a property of the **data flowing through the logic**, not of the logic itself. You cannot read it off the schematic. The identical multiplier burns an order of magnitude more power at 50 % activity than at 5 %, and nothing in its gate-level structure tells you which it will be.

So block-power estimation is, almost in its entirety, **activity estimation**: predicting $\alpha$ for every node, on a workload that resembles the one the chip will really run, *before there is silicon to measure*. Two families of methods fall out of that single need, and they sit at opposite ends of one accuracy-vs-cost axis:

- **Vectored (simulation-driven).** Drive the design with representative input *vectors*, simulate, and count the toggles. Empirical and accurate — but only for the workload you actually ran, and slow.
- **Vectorless (probabilistic).** Assign statistics — a signal probability and a toggle rate — to the primary inputs, and *propagate* them analytically through the logic. No vectors, fast, input-independent — but it must assume the inputs are independent, and real data is not.

Everything below is either a point on that vectored↔vectorless axis or a reason the estimate is hard: the probability algebra of propagation (§3), the correlation that breaks it (§4), glitch power that a zero-delay model cannot even see (§5), and the fidelity ladder from architectural models to gate-level signoff (§7). This page owns *how $\alpha$ is obtained and turned into watts*. The physics of the $CV^2$ per transition lives in [CMOS_Fundamentals §4](../00_Fundamentals/01_CMOS_Fundamentals.md); the chip-level budget these watts feed lives in [Power_Fundamentals](01_Power_Fundamentals.md).

---

## 1. What the activity factor actually is

Before estimating $\alpha$ you have to be precise about what it counts, because two different numbers get called "activity" and they differ by a factor of two.

- **Signal probability** $p$ — the long-run fraction of cycles a node holds logic 1. A static property of the value.
- **Activity factor** $\alpha$ — the *expected number of energy-drawing transitions per clock*. Only the $0\!\to\!1$ transition pulls charge from the supply (it charges the load cap; the $1\!\to\!0$ edge dumps that charge to ground and draws nothing further). Over a full up-then-down cycle the node dissipates $CV^2$, so the $\alpha$ in $P=\alpha C V^2 f$ is the probability of a $0\!\to\!1$ transition per cycle, $\alpha \in [0,1]$.

For a **temporally independent** (memoryless) node the two are tied together:

$$
\alpha \;=\; P(0\!\to\!1) \;=\; p\,(1-p), \qquad \alpha_{\max}=0.25 \ \text{ at } p=0.5
$$

where $p$ = signal probability. This is why "random data" sits at $\alpha \approx 0.25$: uniformly random bits have $p=0.5$ and toggle a quarter of the time on the charging edge. The **toggle rate** that simulators report counts *both* edges and is therefore $2\alpha$; a clock has $\alpha=1$ (one rising edge every cycle) but a toggle rate of 2. Keeping this convention straight is the first thing a power estimate can get quietly wrong — the tool hands you toggle counts, but the equation wants $0\!\to\!1$ charging events.

Activity spans two orders of magnitude across a chip, and the ladder itself is the intuition:

| Signal class | Typical $\alpha$ | Why |
|---|---|---|
| Clock net | $1.0$ | one charging edge every cycle, by definition |
| Datapath, random data | $\approx 0.25$ | the $p(1-p)$ maximum |
| Datapath, real data | $0.05$–$0.25$ | correlated: sign bits, zero bytes, slow-moving MSBs |
| Control / FSM | $0.02$–$0.10$ | mostly stable between events |
| Config / CSR | $<0.01$ | written once, then static — prime clock-gating target |

The gap between "random data" and "real data" is not noise; it is **correlation** (§4), and it is exactly the gap a vectorless estimator has to model or else over-count. Two subtleties already visible here — that clock nets dominate because $\alpha=1$ meets the largest capacitance on the die (§6), and that low-$\alpha$ config nets are where clock gating pays — are the payoff of getting activity right.

**The bridge this page is about.** Architecture teams describe a block by its *utilization* or *occupancy* — the fraction of cycles it issues an instruction, accepts a transaction, or fires a MAC. Power teams need $\alpha$. The block's power model is the translator between them, and mapping occupancy to per-net activity is the single hardest, most valuable step in the whole flow. Everything else is bookkeeping around that conversion.

---

## 2. Two ways to get $\alpha$: vectored vs vectorless

The dichotomy of §0 is the field's fundamental fork, and each side is the other's weakness turned inside out.

| | **Vectored** (simulation-driven) | **Vectorless** (probabilistic) |
|---|---|---|
| Input | representative stimulus (tests, traces, software) | signal probability + toggle rate on primary inputs |
| Mechanism | simulate, record every net's toggles (VCD/SAIF) | propagate probabilities through the netlist (§3) |
| Correlation | captured *for free* — it is implicit in the vectors | must be modeled explicitly, or assumed away |
| Accuracy | high *on the workload run* | typically $10$–$20\%$ worse when correlation matters |
| Cost | expensive: simulation is slow, few cycles | cheap: one pass over the graph, seconds |
| Blind spot | **coverage** — only sees the vectors you chose | **correlation** — independence assumption (§4) |
| Used when | signoff, workload-specific power, peak vectors | early "is any node pathological?" screening, untestable blocks, coverage backfill |

Neither is a superset of the other, so real flows use both: vectorless to sweep the whole design cheaply and flag hot nets that no test happens to exercise, vectored to pin down the number that matters on a workload that matters. The vectored side has its own **accuracy ladder**, set by how realistic the stimulus is:

| Activity source | Typical error | Why |
|---|---|---|
| Analytical / spreadsheet utilization | $\ge 2\times$ | a guess per sub-block from a performance model |
| Architectural-simulator event counts | $30$–$50\%$ | per-block events $\times$ energy-per-event, no real toggles |
| RTL simulation (short directed tests) | $20$–$30\%$ | real toggles, but seconds of activity |
| RTL/gate activity from **emulation** | $10$–$20\%$ | real software: boot, a game frame, an inference pass |
| Gate-level + parasitics, real windows | $5$–$15\%$ | toggles *and* timing (glitch, §5) |

The jump from short tests to emulation is the one that surprises people: a directed testbench keeps a block continuously busy and so **systematically over-estimates** $\alpha$ while **under-estimating** idle and clock-gated residency. Real software has phases — an OS timer tick that keeps a "mostly idle" cluster at 30 % clock activity, a driver bug that never lets a block reach its low-power state — and only long windows see them.

**The worst case is also a vectored question.** Signoff needs not the *average* $\alpha$ but the *peak*, to size the power grid against dynamic IR drop. That peak is found with an adversarial **power-virus** vector — synthetic stimulus that maximizes $\alpha$ everywhere at once (all lanes computing, maximum-toggle data patterns). It is the vectored *upper* bound, in contrast to the average-activity estimate the battery-life model wants; the two live at opposite ends of the same vectored method, and the grid must survive the burst even though cooling only has to handle the average.

### 2.1 The statistics of a finite window

A vectored estimate is a *measurement over a finite sample*, so it carries sampling error, and that error is the mathematical form of the "coverage" blind spot. Treat a node's $0\!\to\!1$ events over $N$ cycles as Bernoulli with rate $\alpha$; the natural estimator $\hat\alpha = (\text{count})/N$ then has

$$
\operatorname{Var}[\hat\alpha] \approx \frac{\alpha(1-\alpha)}{N}, \qquad
\frac{\text{SE}[\hat\alpha]}{\alpha} \approx \sqrt{\frac{1-\alpha}{\alpha\,N}}
$$

where $N$ = simulated cycles, $\alpha$ = true activity, SE = standard error. The relative error scales as $1/\sqrt{\alpha N}$, so **rare-toggling nets are the expensive ones**: a control net at $\alpha=10^{-3}$ needs on the order of $10^5$–$10^6$ cycles just to pin its activity to $\pm 10\%$, which a microsecond directed test cannot supply. This is the quantitative reason low-activity control and configuration logic is estimated badly by short simulation and why emulation (billions of cycles) exists — not to average a busy datapath more finely, but to *see the tail* at all.

Sampling variance is the optimistic half of the story; the pessimistic half is **representativeness**. Even an infinite window of the *wrong* workload converges to the wrong $\alpha$, and the phase-to-phase swing between workloads dwarfs the within-workload noise. Hence the discipline of choosing power-representative windows (the SimPoint idea) rather than simply simulating longer: a well-placed $10\,\mu s$ window beats a poorly-placed millisecond.

---

## 3. Vectorless in theory: propagating probabilities through logic

Vectorless estimation earns its speed by never simulating a vector — it computes each internal node's statistics directly from the inputs'. Given each primary input's signal probability, and *assuming the inputs are independent*, probability flows through the Boolean operators by a small algebra:

$$
\begin{aligned}
\text{NOT: } & p_y = 1-p_a &
\text{AND: } & p_y = p_a\,p_b \\
\text{OR: } & p_y = 1-(1-p_a)(1-p_b) &
\text{XOR: } & p_y = p_a + p_b - 2p_a p_b
\end{aligned}
$$

where $p_a,p_b$ = input signal probabilities, $p_y$ = output. One topological pass yields $p$ at every node; feeding each $p$ into $\alpha = p(1-p)$ gives a first activity estimate.

That last step secretly re-assumes temporal independence, so the rigorous formulation works in **transition density** instead of re-deriving $\alpha$ from $p$. Najm's result propagates activity through a gate via the *Boolean difference*: node $y$ can only toggle when it is *sensitive* to an input that toggled, and it is sensitive to $x_i$ exactly when $\partial y/\partial x_i = y|_{x_i=1}\oplus y|_{x_i=0}$ is true. For independent inputs,

$$
D(y) \;=\; \sum_i P\!\left(\frac{\partial y}{\partial x_i}\right) D(x_i)
$$

where $D(\cdot)$ = transition density (transitions per unit time) and $P(\partial y/\partial x_i)$ = signal probability of the Boolean difference (the fraction of the time $y$ is sensitive to $x_i$). The formula makes the character of each gate quantitative:

- **XOR** ($y=a\oplus b$): $\partial y/\partial a \equiv 1$ — $y$ flips *whenever either input flips*, so $D(y)=D(a)+D(b)$. XOR **passes all activity through**, which is why adders, LFSRs, and crypto datapaths are activity hot-spots.
- **AND** ($y=ab$): $\partial y/\partial a = b$, so $D(y)=p_b\,D(a)+p_a\,D(b)$. AND **attenuates** activity by the probability the *other* input is enabling — an AND with a mostly-0 input is a natural activity filter, the mechanism operand isolation and clock gating exploit.

This is the whole theoretical spine of vectorless estimation: a single graph traversal, gate rules that either pass or attenuate density, and one load-bearing assumption — input independence — that §4 is about to break.

---

## 4. Why the estimate is hard: temporal and spatial correlation

Both the memoryless model $\alpha=p(1-p)$ and the propagation algebra of §3 assume signals are independent — of their own past, and of each other. Real signals are neither, and the two failures have names.

**Temporal correlation — a signal remembers its last value.** Real data has runs. A sign bit stays 0 through a long stretch of positive numbers; the MSBs of an up-counter almost never move while the LSB toggles every cycle. Such a signal can have $p=0.5$ yet an $\alpha$ far *below* the memoryless $0.25$, because "was 0, is 0" is the common case. The memoryless model, which assumes each cycle is an independent coin flip, systematically **over-estimates** the activity of slow-moving data and **under-estimates** bit-flipping counters. The fix is to characterize a signal by both $p$ and $\alpha$ independently (which SAIF does, recording time-at-1 *and* toggle count) rather than deriving one from the other.

**Spatial correlation — signals share a source.** The product rule $p_y=p_a p_b$ is exact only when $a$ and $b$ are independent. At **reconvergent fanout** — where one signal fans out, passes through different logic, and meets itself downstream — they are not, and the error can be large. The textbook trap is $y = a \wedge \bar a$: the true answer is $p_y=0$, but blind propagation gives $p_a(1-p_a)$, a spurious activity out of thin air. Buses are correlated the same way (adjacent bits of an address move together), so treating them as independent nets mis-estimates the whole datapath.

These two effects *are* the accuracy gap in the §2 table. A vectored simulation gets correlation for free — it is baked into the vectors, so the toggles it counts are the real, correlated ones. A vectorless estimator has to model correlation explicitly (pairwise correlation coefficients, supergate/BDD analysis of reconvergent regions) or accept the error, and modeling it fully is as expensive as the simulation it was trying to avoid. That is the fundamental reason vectorless is *fast but approximate* and vectored is *accurate but expensive* — not an implementation detail, but the independence assumption meeting real data.

---

## 5. Glitch power: the activity a zero-delay model can't see

Everything so far counted *functional* transitions — one per node per cycle at most. Real gates also produce **glitches**: spurious extra transitions within a single cycle, and at advanced nodes they are not a rounding error but **25–40 % of dynamic power** in datapath-heavy blocks (higher still in GPU/video-class designs). They are also the hardest activity to predict, for a reason that cuts to the vectored/vectorless divide.

**The mechanism is timing, not logic.** A gate whose inputs derive from a common source receives them with a time spread $\Delta t$ set by path-delay imbalance. If $\Delta t$ exceeds the gate's inertial (switching) delay $\tau$, the output pulses to a wrong value and back before settling — one glitch, drawing a full (or partial) $CV^2$ for a result that was never logically real.

```text
   a ─────\
           XOR ── y     a and b both rise, but b arrives ~80 ps late:
   b ──╲    /           y pulses high for ~80 ps, then falls back —
       (+80ps)          two spurious transitions, ~CV² wasted, no logical change
```

Two properties make glitches grow rather than average out:

- **Amplification with depth.** A glitched output is an input to the next stage, which can glitch again. Through deep reconvergent structures — ripple-carry chains, Wallace-tree multipliers — glitches compound, and a single node can see *more than one* spurious transition per cycle. Arithmetic is the worst offender precisely because it is deep and reconvergent.
- **Filtering weakens as nodes shrink.** A pulse narrower than $\tau$ is swallowed (inertial filtering). But scaling makes gates *faster* (smaller $\tau$) while wire delay spread grows, so each node filters *less* — which is why the glitch share of power has climbed with every process generation rather than staying put.

**Why it defeats RTL power.** A zero-delay RTL simulation collapses every transition to an instant and lets them all happen simultaneously at the clock edge — so it can represent *no* glitch at all. RTL-derived power is therefore a **glitch-free lower bound**, and it under-estimates datapath power systematically. Seeing glitches requires timing, and that reopens the §2 fork:

- **Vectored with timing** — gate-level simulation with SDF-back-annotated delays produces the real glitches, at real (slow) gate-sim speed. This is the only rung that sees them fully, which is why glitch power is fundamentally a *signoff* number.
- **Vectorless with timing** — propagate arrival-time *windows* statistically through the timing graph and estimate expected spurious transitions per node, without vectors. Fast and approximate; the "glitch mode" in RTL/gate power tools. Signoff reports even split the result into *transparent* (full-swing) and *filtered* (partial-swing) glitches, since a partly-formed pulse still burns partial $CV^2$.

The practical consequences flow to reduction, not estimation, and live in [Power_Reduction_Techniques](03_Power_Reduction_Techniques.md): path balancing to equalize arrival times, operand isolation and don't-care gating to stop invalid data rippling through wide datapaths, and retiming to break deep combinational clouds. The device-level view of hazard switching is in [Power_Fundamentals §9](01_Power_Fundamentals.md).

---

## 6. From $\alpha$ to block power: component decomposition

The `sum over nets` picture of $P_{block}$ is conceptually right but is never evaluated net-by-net above gate level. Instead a block is broken into a handful of **component primitives**, each with its own model and — the reason this section belongs here — its own *source* for the three inputs $C$, $\alpha$, and leakage. This is exactly how McPAT and Wattch decompose a block architecturally and how PrimePower/Joules decompose it at RTL.

$$
P_{\text{comp}} = \underbrace{\alpha\,C_{\text{eff}}\,V^2 f}_{\text{or } E_{\text{op}}\times(\text{access rate})} + \underbrace{N_{\text{dev}}\,I_{\text{leak}}(V_t,V,T)\,V}_{\text{static}}
$$

where $C_{\text{eff}}$ = effective switched capacitance (gate + wire), $E_{\text{op}}$ = characterized energy per access, $I_{\text{leak}}$ = per-device leakage. The inputs come from different places for different primitives:

| Component | Dynamic model | Where $\alpha$ (activity) comes from |
|---|---|---|
| Combinational logic | $\alpha\,C_{\text{eff}}V^2 f$ per net, *including glitch* $CV^2$ | net annotation (SAIF) or vectorless propagation (§3) |
| **Flip-flop + clock tree** | clock-pin $CV^2$ **every cycle** ($\alpha_{\text{clk}}=1$) + data-path $CV^2$ at $\alpha_{\text{data}}$ | clock-gating efficiency (CGE) from activity; **clock tree dominates** |
| **SRAM / cache** | per-access read/write energy (**CACTI**: decoder + wordline + bitline + sense-amp) $\times$ access rate | access rate from perf sim or SAIF; leakage/retention often exceeds dynamic |
| Interconnect / wire | $\alpha\,C_{\text{wire}}V^2 f$; long global wires dominate | net annotation or per-mm cap density |
| I/O / PHY | energy per transition, often fixed mW/Gbps per lane | lane utilization from datasheet/IP characterization |

Two primitives concentrate the power, and both are structural rather than workload-driven — which is why they are the first levers:

- **The clock tree is the headline.** Its net toggles every cycle by construction ($\alpha=1$) and it carries the largest fanout and total capacitance on the die (CTS buffers, every FF clock pin, the H-tree). So **clock power is commonly 30–40 % of total dynamic power**, higher in flop-dense, low-logic-depth blocks. This is the whole reason clock gating is the first-line dynamic-power lever ([Power_Reduction_Techniques §2](03_Power_Reduction_Techniques.md)): killing one node's clock removes its single biggest per-cycle energy term, guaranteed, independent of data.
- **Memory arrays concentrate the rest.** [CACTI](https://en.wikipedia.org/wiki/CACTI) is the standard analytical model for array access energy — given (capacity, ports, banks, node) it sums decoder, wordline, **bitline** (the dominant term: precharge and discharge of long bitlines), sense-amp, and H-tree energy. In large arrays *leakage* and, in retention modes, retention power often exceed dynamic, which is why last-level caches are aggressively power-gated or held at low $V$ when idle.

Everything else in the block is combinational logic and wires, whose power the earlier sections were about estimating.

---

## 7. The fidelity ladder: architectural → RTL → gate → SPICE

Because activity gets more real and structure gets more detailed as a design firms up, power estimation is a ladder you descend over the project, trading runtime for fidelity — the same speed↔accuracy ladder as [performance modeling](../01_Architecture_and_PPA/01_Modeling/01_Performance_Modeling_and_DSE.md#1-the-modeling-fidelity-ladder). Each rung needs both a *structural* model (what is instantiated) and an *activity* source (this whole page).

| Level | Example | Accuracy | Speed | Activity source |
|---|---|---|---|---|
| **Architectural** | McPAT, Wattch (+CACTI for arrays) | $\pm 20$–$30\%$ (worse un-calibrated) | instant | event counts from a perf simulator (gem5/Sniper) — *not* real toggles |
| **RTL power** | PrimePower RTL, Joules, PowerArtist | $\pm 10$–$20\%$ vs gates | minutes–hours | RTL-sim SAIF/FSDB or emulation activity |
| **Gate-level** | PrimePower, Voltus | $\pm 5$–$10\%$ (signoff) | hours–days | SDF-annotated gate sim — **the only rung that sees glitch** (§5) |
| **SPICE** | HSPICE, Spectre | golden per-cell | impractical above small blocks | actual transient waveforms |

Two things about this ladder carry the design decisions:

**Why shift left.** Discovering a power bust at gate-level signoff is a schedule disaster, so the industry runs RTL power from the moment RTL exists — its $\pm 10$–$20\%$ accuracy is enough to *trend* and to catch architecture-level regressions even though it cannot sign off. Mature teams run RTL power on fixed workload snippets at every RTL drop with a per-block budget, so a merge that drops clock-gating efficiency from 78 % to 60 % is flagged like a failing test. The vendor tool you use matters far less than feeding it *realistic activity*: the number is only as good as the vectors or the input statistics behind it.

**The calibration chain.** Each rung is anchored by the one below it. SPICE characterizes the `.lib` energy and leakage tables; those feed gate-level signoff; gate-level results calibrate the RTL-power models; RTL and silicon results calibrate the architectural coefficients. An un-calibrated McPAT run can be $2\times$ off — the same trap as an un-validated cycle-accurate performance model — because a bottom-up architectural model is a stack of assumptions with no measurement holding it down. The chain continues past tape-out: in silicon the same weighted-activity idea reappears as an on-die **power proxy** ($\hat P = w_0 + \sum_i w_i\,\text{event}_i$, weights fit to measured power), closing the loop by feeding real activity back to re-anchor the models — but that live estimation, and the closed-loop management it drives, belong to the reduction and signoff pages, not here.

The mechanics of the activity files themselves — SAIF's backward/forward annotation, the toggle/time-at-1 fields, VCD/FSDB capture — are in [Power_Analysis_and_Signoff §2](05_Power_Analysis_and_Signoff.md). Composing these per-block estimates into a full chip, with the contention and DVFS-budget layers that make it more than a sum, is [Full_Chip_Modeling](../01_Architecture_and_PPA/01_Modeling/02_Full_Chip_Modeling.md).

---

## 8. Numbers to memorize

| Quantity | Value | Why (section) |
|---|---|---|
| Activity factor definition | $\alpha=P(0\!\to\!1)=p(1-p)$, memoryless | §1 |
| Random-data $\alpha$ (max of $p(1-p)$) | $0.25$ at $p=0.5$ | §1 |
| Real-data datapath $\alpha$ | $0.05$–$0.25$ (correlation pulls it down) | §1, §4 |
| Clock-net $\alpha$ | $1.0$ (one charging edge/cycle) | §1, §6 |
| Toggle rate vs $\alpha$ | toggle rate $=2\alpha$ (both edges) | §1 |
| Clock-tree share of dynamic power | $30$–$40\%$ (higher in flop-dense blocks) | §6 |
| Glitch share, $\le 7$ nm datapath | $25$–$40\%$ (up to $\sim 60\%$ GPU-class) | §5 |
| Vectorless vs vectored accuracy gap | $\sim 10$–$20\%$ when correlation matters | §2, §4 |
| RTL power accuracy vs gate-level | $\pm 10$–$20\%$ | §7 |
| Gate-level signoff accuracy | $\pm 5$–$10\%$ | §7 |
| Architectural (McPAT) accuracy | $\pm 20$–$30\%$ calibrated ($2\times$ un-cal.) | §7 |
| Finite-window relative error | $\propto \sqrt{(1-\alpha)/(\alpha N)}$ | §2.1 |
| Emulation vs simulation scale | $10^{9}$+ cycles vs $10^{5}$–$10^{6}$ | §2 |
| Density through XOR / AND | XOR passes 100 %; AND attenuates by enabling-input $p$ | §3 |

---

## 9. Cross-references

- **Down the stack (the physics these estimates rest on):** [CMOS_Fundamentals §4](../00_Fundamentals/01_CMOS_Fundamentals.md) (the $\tfrac12 CV^2$ dissipated per transition and the leakage that sets the static term), [Power_Fundamentals](01_Power_Fundamentals.md) (the switching-power derivation, hazard/glitch fundamentals §9, and the total-power equation this page supplies the $\alpha$ for).
- **Up the stack (what consumes the estimate):** [Power_Analysis_and_Signoff](05_Power_Analysis_and_Signoff.md) (SAIF/VCD annotation mechanics and the peak-activity vectors that drive dynamic IR-drop signoff), [Power_Reduction_Techniques](03_Power_Reduction_Techniques.md) (clock gating, operand isolation, and path balancing — what you *do* about the high-$\alpha$ nets and glitches this page finds), [Full_Chip_Modeling](../01_Architecture_and_PPA/01_Modeling/02_Full_Chip_Modeling.md) (composes per-block $\alpha C V^2 f$ into a chip with contention and DVFS layers).
- **Adjacent:** [Performance_Modeling_and_DSE](../01_Architecture_and_PPA/01_Modeling/01_Performance_Modeling_and_DSE.md) (the same fidelity ladder, and the event counts / utilization an architectural power model turns into $\alpha$).

---

## References

1. Rabaey, J.M., Chandrakasan, A., and Nikolić, B., *Digital Integrated Circuits: A Design Perspective*, 2nd ed., Prentice Hall, 2003. The $\alpha C V^2 f$ model and the $p(1-p)$ activity result.
2. Najm, F.N., "Transition Density: A New Measure of Activity in Digital Circuits," *IEEE TCAD*, 12(2), 1993. The Boolean-difference density propagation of §3.
3. Najm, F.N., "A Survey of Power Estimation Techniques in VLSI Circuits," *IEEE TVLSI*, 2(4), 1994. The vectored-vs-vectorless taxonomy and the correlation problem of §4.
4. Li, S. et al., "McPAT: An Integrated Power, Area, and Timing Modeling Framework for Multicore Architectures," *MICRO*, 2009. The architectural component decomposition of §6–§7.
5. Muralimanohar, N., Balasubramonian, R., and Jouppi, N.P., "CACTI 6.0: A Tool to Model Large Caches," HP Labs, 2009. The array access-energy model of §6.
