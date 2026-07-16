# Static Timing Analysis — Proving Timing Without Simulating Vectors

> **Prerequisites:** [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) (the clock, I/O, and exception constraints STA consumes), [PLL_DLL_and_Clock_Distribution](../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md) (where clocks, jitter, and insertion delay come from), CMOS fundamentals (the FO4 delay yardstick).
> **Hands off to:** [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md) (IR-drop → cell-delay coupling), [Signal_Integrity_Reliability](../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md) (crosstalk delta-delay), [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md) (clock-tree synthesis, timing ECO).

---

## 0. Why this page exists

You cannot prove a chip meets timing by simulating it. A modern block has more timing paths than atoms you could ever apply vectors to, the *worst-case* path is data-dependent and you have no way to know which stimulus excites it, and the worst-case *silicon* is a process/voltage/temperature corner you cannot reproduce on your bench. Dynamic timing simulation answers "did these particular vectors, on this particular delay annotation, work?" — a measure-zero sample of the space that must be exhaustively safe.

Static timing analysis inverts the problem. Instead of exhausting **stimuli**, STA exhausts **paths**: for every register-to-register path it statically computes the *latest possible* signal arrival and the *required* time, and reports their difference — the **slack**. It never applies a vector, never toggles a net, and is completely **independent of stimulus**. One analysis certifies that no input pattern, on no path, can violate a flip-flop's timing at the modeled corner.

This page derives that machinery from its purpose. The organizing question for every section is *why must this exist* — why the analysis is graph-based rather than path-enumerated, why setup and hold are the two inequalities they are, why clock skew has a signed effect, why we add margins we know are pessimistic, and why one line of SDC (a multicycle exception) is both the most useful and the most error-prone thing you will write. The reference material — `.lib` table formats, SPEF syntax, PrimeTime report columns — is compressed to the essential; the reasoning is expanded. By the end you should be able to write the slack equation from memory, explain what STA is pessimistic about and what it will happily miss, and get a multicycle hold check right.

---

## 1. The exhaustiveness argument: why static beats simulated

Two facts make simulation the wrong tool for a *timing proof*.

**The path count is astronomical, and the worst path is invisible to you.** A cone of logic with reconvergence has a number of structural paths that grows *exponentially* in its depth (a chain of $k$ 2-input reconvergent stages exposes up to $2^k$ source-to-sink paths). Which one is critical depends on the exact cell sizes, wire loads, and even the switching direction — you cannot guess the vector that sensitizes it. Simulation samples a handful; STA must cover all of them.

**The failure is a "too late" or "too early" *inequality*, not a functional bug.** A path violates timing when its delay is longer (setup) or shorter (hold) than the clock allows — a statement about *delay bounds*, not logic values. You can settle an inequality analytically with worst-case delays; you do not need to observe the transition happen.

So STA reframes timing as a static, vectorless computation over three quantities defined at every point in the design:

- **Arrival time** $a$ — the latest (for setup) or earliest (for hold) a signal can appear at a point, given the worst combination of upstream delays.
- **Required time** $r$ — the latest (setup) or earliest (hold) it is *allowed* to appear so the capturing flop still samples correctly.
- **Slack** $s$ — the margin between them:

$$
\boxed{\,s \;=\; r - a\,}\qquad s \ge 0 \text{ passes},\quad s < 0 \text{ violates.}
$$

The entire tool is a machine for computing $a$ and $r$ everywhere and reporting the minimum slack. Everything below is *how* it computes them (the timing graph, §2), *what relationship defines* $r$ (setup/hold, §3), and *what uncertainty inflates* the worst case (clocking §4, variation §5). Dynamic **gate-level simulation with SDF back-annotation** still has a role — verifying asynchronous logic, reset sequencing, and X-propagation that STA's linear model cannot express — but it certifies *function under one stimulus*, never *timing over all of them*. STA owns the timing proof precisely because it is exhaustive over paths.

---

## 2. The timing graph: exhaustive over paths without enumerating them

If the path count is exponential, how can STA possibly be exhaustive in practice? Because it never enumerates paths. It builds a **timing graph** — a directed acyclic graph whose *nodes* are pins/ports and whose *edges* are **timing arcs** (cell arcs from input to output pin, and net arcs from a driver to each receiver) — and propagates arrival times through the graph, not through paths.

### 2.1 Block-based propagation is dynamic programming over the DAG

Give each arc $u\!\to\!v$ a delay $d(u,v)$. **Actual arrival time** propagates *forward*, taking the worst incoming edge at each node; **required arrival time** propagates *backward*, taking the tightest outgoing constraint:

$$
\text{AAT}(v) \;=\; \max_{u \,\to\, v}\big(\text{AAT}(u) + d(u,v)\big),
\qquad
\text{RAT}(v) \;=\; \min_{v \,\to\, w}\big(\text{RAT}(w) - d(v,w)\big),
$$

with slack $s(v) = \text{RAT}(v) - \text{AAT}(v)$ at *every* node. Launch flops seed AAT with (clock arrival + clock-to-Q); capture flops seed RAT with (capture-edge − setup). The hold analysis is the mirror image — propagate the *earliest* arrival (min in the forward pass) against the earliest required time.

The decisive property: the $\max$ over fan-in at each node **implicitly selects the worst path reaching that node**, so one sweep over the graph's $O(E)$ edges computes the worst arrival at all $O(V)$ nodes. An exponential set of paths collapses into a linear traversal — this is just dynamic programming on a DAG, and it is the entire reason STA scales to $10^8$-gate designs while remaining exhaustive over paths. You get the worst path *for free* by back-tracing the max-choices from the worst-slack endpoint.

Two smaller facts complete the graph model:

- **Arc delay is not a constant** — it depends on the driver's input **slew** (transition time) and the output **load** (fanout pin caps + wire cap from parasitic extraction). The delay model is the oracle that returns $d$: **NLDM** stores delay and output slew as 2-D lookup tables indexed by (input slew, output cap) and threads each stage's output slew into the next arc, so slew and delay co-propagate along the graph. Below ~16 nm the lumped-capacitance assumption breaks (resistive interconnect shielding, waveform distortion from Miller and multi-stage turn-on), so **CCS/ECSM** model the driver as a time-varying current source into the extracted RC network — 2–4× slower and 4–10× larger libraries, but ±2–3 % delay accuracy versus NLDM's ±10–15 %. Parasitics arrive as SPEF from extraction; the RC-to-delay reduction (Elmore as a first-order upper bound, AWE/PRIMA for signoff) lives in [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md).
- **Unateness** decides which polarities an arc pairs. A *positive-unate* arc (buffer, AND) sends rise→rise; a *negative-unate* arc (inverter, NAND) sends rise→fall; a *non-unate* arc (XOR, a clocked latch) depends on side inputs, so the tool must try **both** rise and fall and keep the worse — doubling the arc's analysis. This is bookkeeping, not concept, but it is why a design full of XOR trees analyzes more slowly.

### 2.2 Graph-based vs path-based: the accuracy/runtime trade-off

Block-based (graph-based, GBA) propagation is fast but **pessimistic**, for a structural reason. At a reconvergent node it must keep a *single* worst arrival time and a *single* worst slew — but these can come from **different** fan-in edges. Carrying the worst slew (from one path) onto the worst-time path (from another) over-estimates downstream delay, because no single real path had both. GBA also applies worst-case derating (§5) to every arc without regard to whether one physical path can simultaneously realize all of them.

**Path-based analysis (PBA)** removes exactly this pessimism by re-timing one specific path end-to-end: it recomputes each arc with the *actual* slew that path delivers, applies path-correct derating and common-path credit (§4.3), and so reports a tighter — and physically achievable — slack. The cost is that PBA is $O(\text{path length})$ *per path*, so it cannot be run on the whole design.

The trade-off is therefore a **filter**: run GBA over the entire graph (linear, exhaustive, pessimistic), then run PBA only on the top-$K$ GBA-failing endpoints to recover the paths that were false-failed by slew merging and derate stacking. Real closure flows lean on this hard — a design "failing" by tens of ps in GBA frequently passes in PBA, and choosing to fix it in silicon versus recover it in PBA is a routine signoff decision.

---

## 3. Setup and hold from first principles

Everything so far computes arrival times. The *required* time — the constraint arrivals are checked against — comes from the two inequalities a flip-flop imposes. Derive them from what the flop physically needs.

An edge-triggered flop samples D into a feedback latch at the active clock edge. For the latch to resolve to the correct value, D must be **stable for an aperture around the edge**: it must settle a **setup time** $t_{su}$ *before* the edge (so the sampling node reaches a valid level) and stay put a **hold time** $t_h$ *after* it (so the closing switch fully disconnects the input before D moves). Violate the aperture and the internal node is caught mid-transition; the regenerative pair then resolves only after an unbounded, exponentially-distributed settling time — **metastability**. $t_{su}$ and $t_h$ are just the aperture edges, characterized by the library as the point where clock-to-Q has degraded by a set fraction (typically 10 %). Two failures follow: data arriving *too late* (setup) and data arriving *too early* (hold).

### 3.1 The setup inequality (max-delay)

Setup is a "too slow" check: the data launched this cycle must reach the capture flop before its next edge. Define, all in absolute time from a common clock-source edge at $t=0$:

- $T$ — clock period; the capture edge is at $t=T$.
- $t_{cq}$ — launch flop clock-to-Q delay.
- $t_{comb}$ — combinational data-path delay, Q→D.
- $t_{ci,L},\,t_{ci,C}$ — clock **insertion delays** (network latency) from source to the launch and capture flop clock pins.
- $t_{su}$ — capture flop setup time; $t_{unc}$ — clock uncertainty (jitter + margin, §4).

Latest data arrival at the capture D pin, and the time it is required by:

$$
a_{\max} = t_{ci,L} + t_{cq}^{\max} + t_{comb}^{\max},
\qquad
r = T + t_{ci,C} - t_{su} - t_{unc}.
$$

Subtract, and define **clock skew** $t_{skew} \equiv t_{ci,C} - t_{ci,L}$ (capture insertion minus launch insertion):

$$
\boxed{\,s_{setup} = T + t_{skew} - t_{cq}^{\max} - t_{comb}^{\max} - t_{su} - t_{unc}\,}
$$

Setting $s_{setup}=0$ gives the frequency limit — the *one* number the whole timing sign-off ultimately defends:

$$
T_{\min} = t_{cq}^{\max} + t_{comb}^{\max} + t_{su} + t_{unc} - t_{skew},
\qquad f_{\max} = 1/T_{\min}.
$$

Setup is a **max-delay** check because the failure is lateness: STA uses the *slowest* data path against the *earliest* capture clock. Note the sign of skew — **positive skew helps setup** (a later capture edge grants the data more time).

### 3.2 The hold inequality (min-delay)

Hold is a "too fast" check, and its subtlety is that it is checked against the **same** capture edge that already sampled the *previous* cycle's data (at $t=0$, not $t=T$). The newly launched data must not race around and disturb what the flop is holding from that edge — so it must arrive *after* the hold window closes:

$$
a_{\min} = t_{ci,L} + t_{cq}^{\min} + t_{comb}^{\min},
\qquad
r_{hold} = t_{ci,C} + t_h + t_{unc,h}.
$$

Here the failure is earliness, so slack is arrival-minus-required:

$$
\boxed{\,s_{hold} = t_{cq}^{\min} + t_{comb}^{\min} - t_{skew} - t_h - t_{unc,h}\,}
$$

Two consequences that trip people up. First, **$T$ does not appear** — hold is independent of clock period, so *you cannot fix a hold violation by slowing the clock*; it is a race that must be fixed with delay (buffers on the short path). Second, **skew now subtracts**: positive skew *hurts* hold. Setup and hold pull on skew in opposite directions, which is the entire reason clock skew is a signed, double-edged quantity (§4.1).

Hold is a **min-delay** check: the *fastest* data path against the *latest* capture clock. This is the mirror of setup, and the opposition — max data/early clock for setup, min data/late clock for hold — is exactly what forces the opposite-direction clock derating that CPPR later has to correct (§4.3, §5).

### 3.3 A worked slack, both checks

One 1 GHz path, $T=1.0$ ns. Insertion delays $t_{ci,L}=0.40/0.35$, $t_{ci,C}=0.45/0.38$ (max/min); $t_{cq}=0.08/0.05$; $t_{comb}=0.35/0.15$; $t_{su}=0.05$, $t_h=0.02$; $t_{unc}=0.10$ (setup), $0.03$ (hold).

- **Setup** (max data, early capture clock): $a=0.40+0.08+0.35=0.83$; $r=1.00+0.45-0.05-0.10=1.30$; $s_{setup}=+0.47$ ns. **Pass** with margin.
- **Hold** (min data, late capture clock): $a=0.35+0.05+0.15=0.55$; $r=0.45+0.02+0.03=0.50$; $s_{hold}=+0.05$ ns. **Pass**, but tight — the classic profile: comfortable setup, marginal hold.

The +0.05 ns skew here helped setup and ate into hold, illustrating §3.2 directly.

### 3.4 Recovery and removal

Asynchronous set/reset **deassertion** has the same aperture problem: the release edge must settle before the clock (**recovery**, a setup analog) and stay put after it (**removal**, a hold analog), or the flop's reset release goes metastable. Same inequalities with $t_{su}\!\to\!t_{rec}$, $t_h\!\to\!t_{rem}$. This is why the discipline is *asynchronous assertion, synchronous deassertion* through a reset synchronizer — it converts an unconstrained recovery/removal check into an ordinary setup/hold one.

---

## 4. Clocking realities: skew, jitter, insertion delay, and CPPR

§3 assumed the clock arrives at ideal times. It does not, and the corrections are where most real margin is won or lost.

### 4.1 Skew is signed and double-edged

Skew $t_{skew}=t_{ci,C}-t_{ci,L}$ entered setup with a $+$ and hold with a $-$. So the same physical clock imbalance that *buys* setup slack on a path *spends* hold slack on it. Clock-tree synthesis exploits this deliberately — **useful skew** intentionally delays a capture clock to hand setup margin to a critical stage — but it is a *zero-sum reshuffle*: the borrowed time is taken from the next stage and every affected path's hold must be re-verified. Skew is never "reduced to zero" for free; it is *scheduled*.

### 4.2 Jitter, uncertainty, and the ideal→propagated transition

**Jitter** is cycle-to-cycle movement of the clock edge (PLL phase noise, supply-droop-induced delay modulation from [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md)). STA cannot know the instantaneous edge, so it folds jitter — plus design margin — into **clock uncertainty** $t_{unc}$, subtracted from both setup and hold budgets (that is the $t_{unc}$ term in §3). Before the clock tree exists (**pre-CTS**), the clock is *ideal*: its latency is a single `set_clock_latency` number and uncertainty also absorbs an *estimate* of the not-yet-built skew. After CTS (**post-CTS**), `set_propagated_clock` switches to the real per-flop arrival times, skew becomes explicit per path, and uncertainty shrinks to jitter + a thin margin. **Insertion delay** — the source-to-flop clock latency itself — matters less for its magnitude than for how much of it is *shared*, which is the next point.

### 4.3 CPPR: don't derate the shared clock against itself

Here is where §3's opposite-direction derating creates artificial pessimism. Under on-chip variation (§5) setup derates the launch clock *late* and the capture clock *early*. But the launch and capture clock paths **share a common segment** from the root to their divergence point, built from the *same* physical buffers. That shared chain cannot be simultaneously late (as launch) and early (as capture). Applying opposite derates to it double-counts variation that physically cannot occur.

**Clock-path-pessimism removal** (CPPR, a.k.a. CRPR) credits it back. The credit is the OCV spread on the common segment:

$$
\text{CPPR} = t_{common}^{late} - t_{common}^{early},
$$

added to whichever side the analysis pessimized. Example: a common chain of 256 ps derated ±late/early to 256.8 / 223.2 ps yields **≈ 33.6 ps** recovered — enough to flip a marginal path from fail to pass. CPPR grows with common-path length and derate magnitude, which is one more reason to build clock trees that share insertion delay deep before diverging. It is computed automatically per path-pair; it appears as one line in every timing report and is often 20–40 % of the clock uncertainty. Conceptually it is the direct antidote to the setup/hold OCV split — the pessimism §3 and §5 create on the shared clock, handed back.

---

## 5. Variation and margin: OCV, AOCV, POCV, and corners

STA computes a worst case, but "worst case" over *what*? Silicon delay is not a number — it is a distribution over manufacturing variation, supply, temperature, and aging. Margins exist because we sign off *once*, on models, for *billions* of die we will never measure. The design axis is how *tightly* we model that distribution: loose models are simple and portable but waste slack (over-design, or false failures); tight models recover slack but cost characterization and runtime.

### 5.1 Within-die variation: OCV → AOCV → POCV, and the √N argument

**Flat OCV** multiplies every cell delay by a derate $(1\pm\delta)$ — launch/data late, capture early for setup, reversed for hold. Simple, but consider an $N$-stage path. Flat OCV's late bound is

$$
N\mu(1+\delta) = N\mu + \underbrace{N\mu\,\delta}_{\text{pessimism}},
$$

so its added margin grows **linearly in $N$**. Physical variation does not. If each stage delay is independent with mean $\mu$ and standard deviation $\sigma$, the path delay has mean $N\mu$ and — because *variances* add — standard deviation $\sigma\sqrt{N}$. The true $3\sigma$ late bound is

$$
N\mu + 3\sigma\sqrt{N},
$$

whose margin grows only as $\sqrt{N}$. The ratio of flat-OCV pessimism to real statistical margin therefore *diverges* with depth:

$$
\frac{N\mu\,\delta}{3\sigma\sqrt{N}} \;\propto\; \sqrt{N}.
$$

Deep paths are exactly where flat OCV over-margins most — and at 7 nm and below, where $\delta$ is large, that surplus pessimism will fail otherwise-good silicon. Two refinements close the gap:

- **AOCV (advanced OCV)** replaces the flat derate with a table **indexed by path depth** (and optionally physical span): depth 1 gets the full derate, depth 50 a small one — a lookup approximation of the $\sqrt{N}$ averaging.
- **POCV/SOCV (parametric/statistical OCV)** carries an explicit per-cell $\sigma$ and combines paths in root-sum-square, $\sigma_{path}=\sqrt{\sum_i \sigma_i^2}$ — which *is* the statistical model, and also handles reconvergence and slew statistically rather than by table. It recovers the most slack (tens of ps on deep paths) at the cost of per-cell sigma libraries and statistical propagation.

The trade-off is monotone: flat OCV (one number, most pessimistic) → AOCV (depth tables, characterization effort) → POCV (sigma libraries, most runtime, least pessimism). Advanced nodes force the move rightward because flat OCV's *linear* pessimism is no longer affordable.

### 5.2 Die-to-die variation: PVT corners and MCMM

Variation also spans slow/fast **process**, supply **voltage**, and **temperature** — the **PVT corners**, each a full library characterization. Traditionally setup is worst at slow-slow / low-V / high-T and hold at fast-fast / high-V / low-T, but at advanced nodes **temperature inversion** (cells slower when *cold*, because near-threshold the $V_{th}$ rise beats the mobility gain) breaks that monotonicity, so *both* temperature extremes must be checked for each. **MCMM** (multi-corner multi-mode) is the cross product of corners with functional **modes** (functional / scan / sleep), each combination a **scenario** carrying its own `.lib`, SPEF, and SDC.

This is the corners-vs-runtime trade-off in the raw: scenarios multiply into the tens for an SoC, each a multi-hour STA run, so signoff is a compute-farm job and teams prune aggressively to the *dominating* corner per check. **Statistical STA (SSTA)** promised to replace the corner grid with a single distribution-propagating run, but correlation modeling and tool/library support kept it from mainstream signoff; **POCV** is the pragmatic within-die piece that actually shipped, run *across* a reduced corner set rather than instead of it. Advanced nodes only push the count up (aging/NBTI derating, work-function variation, self-heating add dimensions at N3/N2).

---

## 6. Timing exceptions: false paths and multicycle paths

STA is exhaustive by default: it checks *every* structural path against the single-cycle setup/hold relationship of §3. But not every structural path is a real single-cycle path, and forcing one to meet a one-cycle constraint either wastes area (over-fixing) or reports a phantom violation. **Timing exceptions** are the SDC statements that tell STA the true intent (see [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md)). They are *consumed, not derived* — STA trusts them completely, so a wrong exception silently hides a real violation. Two matter most.

### 6.1 False paths — "never check this"

A **false path** is a structural connection that carries no single-cycle capture relationship you care about:

```tcl
set_false_path -from [get_cells cfg_reg*]        ;# static config, written once at boot
set_false_path -through [get_pins dft_mux/TEST]  ;# test-only path, dead in functional mode
set_clock_groups -asynchronous \
    -group {sys_clk} -group {pixel_clk}          ;# unrelated domains: false-path all crossings at once
```

Marking a path false removes it from analysis entirely. The risk mirrors the benefit exactly: a *genuine* asynchronous crossing wrongly declared false is a silicon metastability bug STA will never flag. That is why clock-domain crossings belong to a synchronizer + `set_clock_groups` discipline, not a hand-written `set_false_path` on a data path.

### 6.2 Multicycle paths — the derivation, done correctly

A **multicycle path** is one where the designer *guarantees* — via an enable, a divided clock, or a known-slow datapath — that the capture flop only samples every $N$ cycles, so the data legitimately has $N$ periods to settle. Relaxing *setup* to $N$ periods is the easy half. The **hold** half is where this is routinely, and was previously on this page, gotten backwards. Derive it from the edges.

Number the capture-clock edges after the launch edge $E_0$ (at $t=0$): $E_1$ at $T$, $E_2$ at $2T$, …, $E_k$ at $kT$.

**Single-cycle baseline.** With no exception STA pairs two checks whose edges are *one cycle apart*:

- **Setup** captures at $E_1$: late data must arrive before $E_1 - t_{su}$.
- **Hold** captures at $E_0$: early data must not corrupt the sample the flop takes at $E_0$, so it must arrive after $E_0 + t_h$.

The hold edge sits **exactly one cycle before the setup edge**. That coupling is the invariant PrimeTime preserves, and the whole key to getting multicycle right.

**Apply a setup multiplier.** `set_multicycle_path N -setup` moves the setup-capture edge to $E_N$ ($N$ periods of budget). By the one-cycle-before invariant, PrimeTime's **default hold edge moves right along with it**, to $E_{N-1}$ — *not* to the launch edge:

$$
E_{setup} = E_N \;(t=NT),
\qquad
E_{hold}^{\text{default}} = E_{N-1} \;(t=(N-1)T).
$$

This default hold check is almost always **wrong for the intent**. It demands that data launched at $E_0$ not arrive until after $(N-1)T + t_h$ — i.e., that the datapath be at *least* $(N\!-\!1)$ periods **slow**. Nothing guarantees that; the cone is usually fast. The tool "fixes" the phantom violation by stuffing $(N\!-\!1)T$ of hold buffering into the path, destroying the very setup budget the exception created.

**The fix: pull the hold edge back to the launch edge.** The physically correct hold check is the *same* one a single-cycle path uses — protect the sample at $E_0$ — because the enable means the previously captured value need only stay valid until the flop next samples at $E_N$, and the only edge the new launch can actually violate is $E_0$. So the hold edge must return to $E_0$. The hold multiplier counts how many cycles to pull the default hold edge **back**:

$$
E_{hold} = E_{\,(N-1)\,-\,M_{hold}},
\qquad
\text{set } M_{hold} = N-1 \;\Longrightarrow\; E_{hold} = E_0.
$$

Hence the standard idiom for a setup multiplier $N$ — the two lines always travel together:

```tcl
set_multicycle_path N       -setup -from [get_pins src/Q] -to [get_pins dst/D]
set_multicycle_path (N-1)   -hold  -from [get_pins src/Q] -to [get_pins dst/D]
```

Worked for $N=2$ at $T=2$ ns (a 2-cycle path):

| Check | Edge | Time | Meaning |
|---|---|---|---|
| Setup (`-setup 2`) | $E_2$ | 4 ns | data gets two periods ✓ |
| Hold **default** (no `-hold`) | $E_1$ | 2 ns | phantom: data must arrive after $2\text{ns}+t_h$ ✗ forces huge buffering |
| Hold with `-hold 1` | $E_0$ | 0 ns | correct: ordinary $t_h$ after launch ✓ |

The general edge placement, defaults included:

$$
E_{setup} = M_{setup}\,T,
\qquad
E_{hold} = (M_{setup} - 1 - M_{hold})\,T,
$$

with $M_{setup}=1,\,M_{hold}=0$ reproducing the single-cycle pair $E_1/E_0$.

**The load-bearing correction** versus the previous version of this page: under a setup-only multicycle, PrimeTime's *default* hold edge lands at **$E_{N-1}$** (one cycle before the setup edge), **not** at the launch edge; and `set_multicycle_path (N-1) -hold` moves it **back** to the launch edge $E_0$ — it does *not* move it *forward*. Two footnotes: PrimeTime actually evaluates the tightest of the hold relationships around adjacent edges, but for a single same-frequency clock that reduces to $E_{N-1}$; and for cross-frequency multicycle (slow↔fast) the multiplier is counted on the *faster* clock and the analogous $-hold$ correction still applies. Never use a multicycle exception to paper over an *asynchronous* crossing — MCP presumes a fixed phase relationship; async crossings are false paths / clock groups (§6.1).

---

## 7. Signoff and closure

Signoff is the discipline of running the analysis above across the full scenario set and driving every slack non-negative. The load-bearing ideas:

- **What STA inherits from the clock tree.** STA does not build the tree; it consumes it. Post-CTS it expects skew < 50–100 ps, insertion delay ~0.5–1.5 ns, clock-net slew < 150–250 ps, and low insertion-delay variation — quantities that feed the OCV margins of §5 and the CPPR credit of §4.3. Tree topology, buffering, and shielding live in [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md); clock *generation* and jitter in [PLL_DLL_and_Clock_Distribution](../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md).
- **The setup↔hold fixing tension.** Setup is fixed by making the *max* path faster (upsize cells, swap to lower-$V_t$, restructure logic) or by borrowing skew; hold by making the *min* path slower (insert delay buffers). Because they pull oppositely on both delay and skew, an over-aggressive setup fix (e.g. useful skew, clock buffering) can *create* hold violations elsewhere — every ECO must be re-checked for both, at all corners. Timing ECOs are the targeted post-signoff form of these fixes.
- **Crosstalk delta-delay.** A switching aggressor couples charge onto a victim net, adding or subtracting delay depending on relative switching direction; SI-aware STA computes a per-net delta-delay using aggressor *timing windows* to avoid assuming every aggressor aligns. This can be tens of ps on tightly-pitched advanced-node nets and is part of signoff — the mechanism and fixes are in [Signal_Integrity_Reliability](../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md).
- **Clock-gating checks.** An integrated clock-gating cell's enable must be stable around the clock's *inactive* edge (falling edge for an AND-based gate) or a runt pulse reaches the gated domain. STA models this as an ordinary setup/hold-style check on the enable pin; hold is the dangerous side because the enable path is often short.
- **The signoff bar.** At *every* scenario: setup, hold, recovery, removal slacks ≥ 0; max-transition, max-capacitance, and min-pulse-width clean; clock-gating checks clean; no SI/noise failures. A single negative slack at a single corner blocks tape-out.

---

## Numbers to memorize

| Quantity | Value | Why it matters |
|---|---|---|
| FO4 delay (N5) | ~12–15 ps | gate-delay yardstick; cycle budget in FO4 |
| Setup time (typical DFF, N5) | 30–60 ps | consumes cycle time directly (§3.1) |
| Hold time (typical DFF, N5) | 20–40 ps; can be **negative** | negative hold is a library gift that eases hold closure (§3.2) |
| Clock uncertainty (post-CTS) | ±20–50 ps | jitter + margin; tightens **both** setup and hold (§4.2) |
| Clock skew target | < 50–100 ps | signed: helps setup, hurts hold (§4.1) |
| Insertion delay | ~0.5–1.5 ns | its *shared* part sets the CPPR credit (§4.3) |
| CPPR credit (typical) | 20–40 % of uncertainty | recovered shared-clock pessimism (§4.3) |
| Flat OCV derate | ±5–15 % (rises with node) | linear-in-depth pessimism → over-margins deep paths (§5.1) |
| POCV per-cell sigma | 2–5 % of mean | adds in quadrature (√N), least pessimistic (§5.1) |
| MCMM scenarios (SoC) | ~10–48 | corners × modes; each a multi-hour run (§5.2) |
| Multicycle rule | `-setup N` ⇒ `-hold (N−1)` | pulls default hold from $E_{N-1}$ back to $E_0$ (§6.2) |
| Crosstalk delta-delay | up to tens of ps | SI-aware signoff at tight pitch (§7) |
| DRAM/clock-tree scale | period 0.3–2 ns (0.5–3 GHz) | sets the slack budget being defended (§3.1) |

**The one equation:** $s_{setup} = T + t_{skew} - t_{cq}^{\max} - t_{comb}^{\max} - t_{su} - t_{unc}$, and its hold mirror $s_{hold} = t_{cq}^{\min} + t_{comb}^{\min} - t_{skew} - t_h - t_{unc,h}$ — setup carries $T$ and $+t_{skew}$; hold carries neither $T$ nor a $+$ on skew.

---

## Cross-references

- **Down the stack (what STA is built from):** [Constraints_SDC](../04_Synthesis/02_Constraints_SDC.md) (clocks, I/O delays, and the false-path/multicycle exceptions STA consumes in §6), [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md) (clock-tree synthesis STA inherits in §4/§7, parasitic RC extraction behind the §2 delay model, timing ECOs), [PLL_DLL_and_Clock_Distribution](../03_Frontend_RTL_and_Verification/05_PLL_DLL_and_Clock_Distribution.md) (clock generation, jitter, and insertion delay feeding §4).
- **Up the stack (what STA feeds and depends on):** [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md) (IR-drop → cell-delay coupling that widens the §5 margins), [Signal_Integrity_Reliability](../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md) (crosstalk delta-delay folded into §7 signoff).
- **Adjacent:** DFT/scan mode is one of the MCMM *modes* of §5.2; the ROB/pipeline that these flops implement is upstream in the architecture stack.

---

## References

1. J. Bhasker and R. Chadha, *Static Timing Analysis for Nanometer Designs: A Practical Approach*, Springer, 2009. Graph-based propagation, OCV/CRPR, exceptions.
2. Synopsys, *PrimeTime User Guide* — multicycle-path and hold-multiplier semantics (§6.2), CPPR, POCV.
3. S. Sapatnekar, *Timing*, Springer, 2004. The timing graph and block-based vs path-based analysis.
4. C. Visweswariah et al., "First-Order Incremental Block-Based Statistical Timing Analysis," *DAC*, 2004. Statistical/parametric variation modeling behind §5.
