# Hardware Interview Questions — Worked Problems and Snap Answers by Domain

> Companion to [RTL_Coding_Questions](08_RTL_Coding_Questions.md) (live coding). This page: the *numeric* problems (timing, power, cache, FIFO (First-In First-Out) math) with full solutions, plus the snap-answer bank per domain. Sources for each derivation are linked per section.

---

## 0. Why this page exists

Beyond RTL (Register-Transfer Level) coding, hardware loops test whether you can *compute* under pressure: a setup-slack number, a cache AMAT (Average Memory Access Time), a power saving, an MTBF (Mean Time Between Failures) exponent — and whether your one-sentence definitions are precise. This page drills both: each section opens with the standard worked problems (do them on paper), then the rapid-fire Q&A with the answers an interviewer is fishing for. Numbers are chosen to be representative of N5–N3-era designs.

---

## 1. Static timing

### 1.1 Worked problems

**P1 — max frequency.** FF1 → logic → FF2: $t_{cq}$ = 60 ps, logic = 480 ps, setup $t_{su}$ = 50 ps, clock skew (capture − launch) = +30 ps, jitter margin 20 ps. Max clock?

$$
T_{\min} = t_{cq} + t_{logic} + t_{su} - t_{skew} + t_{jit} = 60+480+50-30+20 = 580\ \text{ps} \Rightarrow f_{\max} \approx 1.72\ \text{GHz}
$$

Positive capture skew *helps* setup (capture clock arrives later). Know the sign convention cold.

**P2 — hold check, same path.** Hold $t_h$ = 40 ps, min $t_{cq}$ = 45 ps, min logic = 25 ps, same +30 ps skew.

$$
t_{cq,\min} + t_{logic,\min} \ge t_h + t_{skew} \;\;?\;\; 45+25 = 70 \ge 40+30 = 70 \;\to\; \text{slack } 0\ \text{ps (marginal!)}
$$

The same skew that bought setup margin *eats* hold margin — the seesaw every STA (Static Timing Analysis) interview circles. Fix: hold buffers on the data path (delay insertion), never by slowing the clock (hold is frequency-independent — say this unprompted).

**P3 — multicycle.** A 64-bit multiplier takes 1.4 ns; clock is 1 GHz. Declare `set_multicycle_path 2 -setup`; what else must you do and why?

*Answer:* `set_multicycle_path 1 -hold` (move the hold check back to the launch edge). Default hold for MCP-2 (multicycle-path) setup checks against edge N+1, demanding the data be stable a full extra cycle — exploding hold fixing for no reason. Also: enable logic must guarantee the destination samples only every 2nd cycle (the constraint documents intent; RTL must enforce it). Derivations: [STA](../06_Signoff/01_STA.md).

**P4 — OCV (On-Chip Variation) flavor.** Why did the industry move OCV → AOCV (Advanced On-Chip Variation) → POCV/LVF? *Answer:* flat OCV derates everything by worst-case % → hopeless pessimism on deep paths; AOCV: derate as f(depth, distance) — random variation averages over long chains ($\sigma_{chain} \propto \sqrt{n}$ not $n$); POCV/LVF (parametric OCV / Liberty Variation Format): per-cell delay distributions (σ moments) propagated statistically. Same physics, decreasing pessimism, increasing signoff cost.

### 1.2 Snap answers

- **Setup vs hold in one breath:** setup = data must arrive before capture edge (limits $f_{max}$, checked at max delay); hold = data must stay stable after the edge (frequency-independent, checked at min delay; violations are silicon-fatal — can't slow-clock around them).
- **Time borrowing:** latches are level-sensitive → data arriving late simply eats into the transparent window of the next stage; flops can't borrow. Useful-skew is the flop-world equivalent (intentional CTS (Clock Tree Synthesis) skew).
- **CRPR (Clock Reconvergence Pessimism Removal):** launch and capture share clock-tree cells; worst-casing one slow and one fast double-counts variation on the *common* segment — CRPR adds back the common-path pessimism.
- **False path vs multicycle:** false = never functionally exercised (CDC (clock domain crossing), static config); multicycle = exercised but allowed N cycles. Misusing false-path on a real path = silent silicon bug; the most dangerous SDC (Synopsys Design Constraints) line.

---

## 2. CDC and metastability

### 2.1 Worked problem — MTBF

2 GHz receiver, async data toggling at 200 MHz, $T_0$ = 100 ps, $\tau$ = 15 ps, resolution time with one sync FF (flip-flop) $t_r$ = 350 ps. MTBF one-flop vs two-flop?

$$
\text{MTBF} = \frac{e^{t_r/\tau}}{T_0 f_{clk} f_{data}}, \quad
\text{1FF: } \frac{e^{350/15}}{100\text{ps} \cdot 2\text{GHz} \cdot 200\text{MHz}} = \frac{e^{23.3}}{4\times10^{7}} \approx \frac{1.3\times10^{10}}{4\times10^{7}} \approx 330\ \text{s}
$$

Two-flop adds a full period (500 ps) of resolution: $e^{(850/15)} = e^{56.7} \approx 4\times10^{24}$ → MTBF ≈ $10^{17}$ s ≈ 3 billion years. **The exponential is the entire story** — each added τ-multiple multiplies MTBF by e.

### 2.2 Snap answers

- **Why can't you synchronize a multi-bit bus with 2FF per bit?** Bits resolve independently → mixed old/new word. Use gray (unit-distance counters only), req/ack + quasi-static data, or async FIFO.
- **Convergence/fan-in rule:** two separately-synchronized signals re-converging in logic can be skewed by a cycle → never combine synchronizer outputs combinationally without a qualifier.
- **What does a CDC tool actually check?** Structural: every crossing has a recognized scheme; glitch sources feeding sync (comb logic before sync FF is illegal); reconvergence; gray-coding on multi-bit; reset domain crossings (the 2025+ added focus — RDC).
- Full schemes: [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md); pulse/toggle code in [RTL_Coding_Questions](08_RTL_Coding_Questions.md) §9.

---

## 3. Power

### 3.1 Worked problems

**P1 — dynamic power.** Block: 2 GHz, $C_{sw}$ = 1.8 nF effective switched cap per cycle at α=1; activity α = 0.12, $V$ = 0.75 V. Dynamic power?

$$
P = \alpha C V^2 f = 0.12 \times 1.8\,\text{nF} \times 0.5625 \times 2\,\text{GHz} \approx 243\ \text{mW}
$$

**P2 — DVFS (Dynamic Voltage and Frequency Scaling).** Same block can finish the workload at 1.2 GHz. Voltage scales ~linearly with f in this range: V = 0.55 V at 1.2 GHz. Energy ratio per task?

Energy/task ∝ $CV^2$ (f cancels for fixed work): $(0.55/0.75)^2 = 0.54$ → **46% energy saving**, plus leakage × longer runtime as the counterweight — race-to-idle vs DVFS depends on leakage fraction. State both terms; that's the senior answer.

**P3 — clock gating.** Clock tree = 30% of block dynamic power; gating achieves 85% idle coverage on a block idle 60% of the time. Saving?

Clock-tree saving = 0.30 × 0.60 × 0.85 ≈ 15.3% of block dynamic power; plus downstream flop-internal clock power and killed datapath toggles (often another ~5–10%). Cheapest power knob in the book — and why ICG (Integrated Clock Gating) insertion is automatic in synthesis ([Power_Reduction_Techniques](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md)).

### 3.2 Snap answers

- **Power gating vs clock gating:** clock gating kills dynamic only (state retained, instant wake); power gating kills leakage too (state lost or retention flops; µs-scale wake, rush-current management, isolation cells on outputs).
- **Why isolation cells?** Floating outputs from an off domain drive X/crowbar current into on domains — clamp at the boundary ([UPF_Power_Intent](../02_Power_and_Low_Power/04_UPF_Power_Intent.md)).
- **Level shifters:** any signal crossing voltage domains; missing LS = silent timing/functional hazard, caught only by UPF (Unified Power Format)-aware checks.
- **Where does leakage go at low Vt (threshold voltage) / high temp?** Subthreshold leakage ∝ $e^{-V_t/nkT}$ — exponential in both; hence multi-Vt mixing (LVT only on critical paths) as a standard closure move.

---

## 4. CPU architecture and memory hierarchy

### 4.1 Worked problems

**P1 — AMAT.** L1: 4-cycle hit, 95% hit; L2: 14-cycle, 80% (local); memory 120 cycles. 

$$
\text{AMAT} = 4 + 0.05 \times (14 + 0.20 \times 120) = 4 + 0.05 \times 38 = 5.9\ \text{cycles}
$$

Follow-up they always add: which helps more, halving L1 miss rate or halving memory latency? Halve L1 miss: 4 + 0.025×38 = 4.95. Halve mem: 4 + 0.05×26 = 5.3. **L1 miss rate wins here** — run the numbers, don't intuit.

**P2 — cache geometry from address bits.** 64 KiB, 4-way, 64 B lines, 48-bit PA: offset = log2(64) = 6 b; sets = 64Ki/(4×64) = 256 → index 8 b; tag = 48−14 = 34 b. Tag SRAM (Static Random-Access Memory) ≈ sets × ways × (34 + state ~3 b) ≈ 256×4×37 b ≈ 4.6 KiB. **VIPT (Virtually-Indexed Physically-Tagged) constraint:** index+offset (14 b) > 12-bit page offset → 2 bits of index are virtual → alias risk: forbid (page coloring), or restrict to ≤ 16 KiB/way, or dual-lookup. This exact trap appears constantly ([TLB_and_Virtual_Memory](../01_Architecture_and_PPA/08_TLB_and_Virtual_Memory.md)).

**P3 — speedup accounting.** 5-stage in-order, CPI (Cycles Per Instruction) contributions: base 1.0 + 0.25 load-use×1 + branch: 18% branches, 70% predicted, 3-cycle flush. CPI = 1 + 0.25 + 0.18×0.3×3 = 1.41. Doubling predictor accuracy to 85%: CPI = 1 + 0.25 + 0.081 = 1.33 → 6% perf. Now they ask: why does the same predictor change matter ~3× more on a 14-stage OoO (Out-of-Order)? Deeper flush (≈14–20 cycles) and wider issue multiply the lost-slot cost: 0.18×0.15×16×(IPC 4, i.e. 4 instructions per cycle) — misprediction cost scales with width × depth ([Branch_Prediction_Deep_Dive](../01_Architecture_and_PPA/06_Branch_Prediction_Deep_Dive.md)).

### 4.2 Snap answers

- **ROB (Reorder Buffer) vs issue queue:** ROB = in-order retirement window (precise exceptions, rename reclaim); IQ = out-of-order wakeup/select window. Sizes decouple: ROB ~300+, IQ ~100− because IQ is CAM (content-addressable-memory)-expensive per entry ([OoO_Execution](../01_Architecture_and_PPA/05_OoO_Execution.md)).
- **Why physical register file rename over ROB-value rename?** Values written once, read from one place; no retirement copy; supports wide machines — cost: free-list management and a level of indirection.
- **MESI (Modified/Exclusive/Shared/Invalid): why is E worth it?** Silent E→M on private write (no bus transaction) — the common single-threaded case writes without coherence traffic ([Cache_Microarchitecture](../01_Architecture_and_PPA/07_Cache_Microarchitecture.md), [ACE_and_CHI](../01_Architecture_and_PPA/12_ACE_and_CHI.md)).
- **Store-to-load forwarding hazard:** load must check older stores in LSQ (load-store queue; address overlap, partial overlap → stall or replay); memory disambiguation prediction (e.g., store-set) lets loads bypass *predicted-independent* stores with replay on violation.
- **Inclusive vs exclusive LLC (Last-Level Cache):** inclusive = snoop filter for free, wastes capacity (duplicates), back-invalidation pathology; exclusive = max capacity, needs separate snoop filter — the AMD/Intel historical split.

---

## 5. Implementation (synthesis → signoff)

### 5.1 Worked problem — the closure narrative

"Your block misses setup by 80 ps at signoff. Walk the fix ladder, cheapest first."

1. **Confirm it's real**: check the path — false/multicycle candidate? SI-induced delta? CRPR applied? An un-annotated constraint costs nothing to fix.
2. **Netlist-local**: upsize/swap-to-LVT cells on the critical arc, legalize-incremental — hours.
3. **Placement/CTS**: pull cells together, useful skew (steal margin from the slack-rich downstream stage) — a day.
4. **RTL micro-restructure**: balance the cone (move logic across the register), retime — days + reverify.
5. **Pipeline stage / spec change**: the honest answer when 80 ps is really 300 ps at the next corner — weeks, architecture sign-off.

Naming the *verification cost* of each rung is what separates senior answers ([Synthesis_and_Optimization](../04_Synthesis/01_Synthesis_and_Optimization.md), [Physical_Design](../05_Backend_Physical_Design/01_Physical_Design.md)).

### 5.2 Snap answers

- **Why does the tool report 0 violations at synthesis and hundreds at PnR (Place-and-Route)?** Synthesis timed with wireload/virtual route; PnR has real RC (resistance-capacitance) + congestion detours + CTS skew + SI (signal integrity). Synthesis numbers are a *promissory note*.
- **Scan insertion's timing cost:** mux in front of every D pin (one mux delay on all reg-reg paths) + hold fixing on scan chains; ~2–5% area ([DFT_and_ATPG](../06_Signoff/02_DFT_and_ATPG.md)).
- **LEC (Logic Equivalence Checking) vs simulation:** equivalence checking proves netlist ≡ RTL for all inputs (combinational induction over mapped state points) — no vectors; required after every netlist surgery (scan, CTS buffers, ECO (Engineering Change Order)) ([Formal_Verification](../03_Frontend_RTL_and_Verification/12_Formal_Verification.md)).
- **Antenna violation in one line:** charge collected on long metal during etch discharges through thin gate oxide — fixed by layer-hopping or antenna diodes; purely a manufacturing-flow effect, invisible in RTL.

---

## 6. Verification

### 6.1 Snap answers

- **Directed vs constrained-random economics:** directed = linear effort per scenario, CR = front-loaded env cost then coverage-per-cycle compounding; the crossover argument is *the* verification-strategy interview ([Verification_Planning_and_Coverage_Closure](../03_Frontend_RTL_and_Verification/11_Verification_Planning_and_Coverage_Closure.md), [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md)).
- **When formal over simulation:** control-dominated, bounded-state, completeness-critical: arbiters (starvation), FIFOs (no overflow), CDC structure, register access policies, security/lockstep — exhaustive where sim samples ([Formal_Verification](../03_Frontend_RTL_and_Verification/12_Formal_Verification.md)).
- **Coverage closure ≠ done:** code+functional coverage at 100% with a wrong/missing covergroup still ships bugs; review coverage *model* against spec, not just the percentage.
- **"Your test passes but the bug exists" — name mechanisms:** checker disabled/X-optimism masking, scoreboard compares only fields that match, assertion off during reset window where bug fires, seed luck — the question tests epistemic humility about green dashboards.

### 6.2 Worked problem — assertion writing under pressure

"Write: after `req` rises, `gnt` must rise within 1–8 cycles, and `req` must hold until `gnt`."

```verilog
property p_handshake;
  @(posedge clk) disable iff (!rst_n)
  $rose(req) |-> req throughout (##[1:8] $rose(gnt));
endproperty
assert property (p_handshake);
cover property (@(posedge clk) $rose(req) ##8 $rose(gnt)); // boundary hit
```

Talking points: `throughout` for the hold condition, `disable iff` reset semantics, antecedent-vacuity (cover the trigger!), and the matching `cover` for the boundary case ([Assertions_and_Coverage](../03_Frontend_RTL_and_Verification/09_Assertions_and_Coverage.md)).

---

## 7. Cross-domain "design a block" prompts (the 30-minute kind)

For each, the expected skeleton — practice narrating these:

1. **Design a 4-master memory arbiter with QoS (Quality of Service).** Clarify: latency vs bandwidth guarantees? → per-master req FIFOs, weighted-RR (weighted round-robin) credit arbiter + aging promotion (no starvation), grant pipelining (hide arbitration behind data), backpressure story, then: how weights map to bandwidth fractions, worst-case latency bound proof ([Network_on_Chip](../01_Architecture_and_PPA/13_Network_on_Chip.md) §6 QoS logic).
2. **Design the fetch stage of a 2-wide CPU.** I$ + ITLB (instruction translation-lookaside buffer) lookup, BTB/RAS/predictor in parallel, fetch-buffer decoupling, misalignment across cache lines, redirect plumbing and bubble math ([Branch_Prediction_Deep_Dive](../01_Architecture_and_PPA/06_Branch_Prediction_Deep_Dive.md) §fetch).
3. **Design a DMA (Direct Memory Access) engine.** Descriptor format (linked list), prefetch of next descriptor during current transfer, outstanding-read tracking (tags), reorder buffer or strict-order choice, completion interrupts + write-coalescing, error/abort semantics, AXI (Advanced eXtensible Interface) burst legality (4 KB) ([AHB_AXI_APB](../01_Architecture_and_PPA/11_AHB_AXI_APB.md)).
4. **Size and design an L2 prefetcher.** Stream detection (delta-correlation), training table geometry, throttling by accuracy/bandwidth headroom, page-boundary stop, interaction with MSHR (Miss Status Holding Register) occupancy ([Cache_Microarchitecture](../01_Architecture_and_PPA/07_Cache_Microarchitecture.md)).

The grading axis is identical in all four: **clarify constraints → block diagram → the one hard sub-problem in depth → verification plan unprompted.**

---

## 8. Numbers bank (cross-domain)

| Quantity | Value | Domain |
|---|---|---|
| FO4 delay @ N5-class | ~8–12 ps | timing intuition |
| Flop $t_{cq}$ / $t_{su}$ | ~50–80 / 30–60 ps | path budgets |
| 2-FF sync MTBF scaling | × $e^{T/\tau}$, τ ≈ 10–20 ps | CDC |
| SRAM access (L1-class macro) | ~0.3–0.5 ns | hierarchy |
| Dynamic power | $\alpha C V^2 f$ | the formula |
| Leakage vs Vt | ~10× per 100 mV Vt drop | multi-Vt |
| Clock tree share of dynamic | 25–40% | why ICG |
| Scan overhead | 2–5% area, 1 mux on D | DFT |
| AMAT | $h + m \times \text{penalty}$, nested | hierarchy |
| Mispredict cost (big OoO) | 15–20 cycles × width | branch econ |
| VIPT safe bound | way size ≤ page size | L1 design |
| Hold fix | data-path delay only, never clock | STA reflex |

---

## Cross-references

- Live-coding companion: [RTL_Coding_Questions](08_RTL_Coding_Questions.md).
- Per-domain depth: [STA](../06_Signoff/01_STA.md), [Power_Reduction_Techniques](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md), [OoO_Execution](../01_Architecture_and_PPA/05_OoO_Execution.md), [Cache_Microarchitecture](../01_Architecture_and_PPA/07_Cache_Microarchitecture.md), [Async_Design_and_CDC](../03_Frontend_RTL_and_Verification/06_Async_Design_and_CDC.md), [UVM_Methodology](../03_Frontend_RTL_and_Verification/10_UVM_Methodology.md), [Formal_Verification](../03_Frontend_RTL_and_Verification/12_Formal_Verification.md).
- AI-systems counterpart (*Common Interview Questions*, *System Design Interview*): companion AI-infra notebook.
