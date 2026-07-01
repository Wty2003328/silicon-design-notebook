# Block Activity Power -- Estimation, Monitoring, and Management

The phrase "block activity power" covers one core idea at three different stages of a
chip's life:

```
P_block = sum over nets ( alpha * C * Vdd^2 * f )  +  P_leak(block)

A block's power is dominated by HOW ACTIVE it is (alpha, utilization),
not just by what's instantiated. The same multiplier burns 10x different
power at 5% vs 50% utilization.

Stage 1 (design time):   PREDICT per-block power from estimated activity
Stage 2 (verification):  MEASURE activity on real workloads, refine
Stage 3 (silicon):       MONITOR activity live and CONTROL power with it
```

This note covers all three. Cross-reference: [Power_Fundamentals](Power_Fundamentals.md)
(activity factor physics), [Power_Analysis_and_Signoff](Power_Analysis_and_Signoff.md)
(SAIF/VCD annotation mechanics), [Power_Reduction_Techniques](Power_Reduction_Techniques.md)
(what to do about high activity).

---

## 1. Activity Factor by Signal Class -- The Starting Intuition

| Signal class | Typical alpha (0->1 per cycle) | Notes |
|--------------|-------------------------------|-------|
| Clock nets | 1.0 | Toggles every cycle by definition (2 transitions = 1 full cycle) |
| Clock enables / gated clock | = CGE-dependent | The whole point of clock gating |
| Datapath (random data) | 0.25-0.5 | Random bit flips with p=0.5 give alpha=0.25 |
| Datapath (real data) | 0.05-0.25 | Real data is correlated: sign bits, zero bytes |
| Control / FSM | 0.02-0.10 | Mostly stable between events |
| Config / CSR registers | <0.01 | Written rarely; prime clock-gating targets |
| Reset, scan enable | ~0 | Static in functional mode |
| Memory bitlines/wordlines | Access-rate dependent | Driven by transaction rate, not clock |

**Key derived metric -- block activity ratio:** for a block, the fraction of cycles it
performs useful work (issues an instruction, accepts a transaction, computes a MAC).
Architecture teams call this *utilization*; power teams convert it to alpha through the
block's power model. The two communities meeting at this number is exactly where
"block activity power" questions come from in interviews.

---

## 2. Design-Time: Per-Block, Per-Mode Power Modeling

### 2.1 The Mode/Block Power Matrix

Every serious SoC maintains a power model long before RTL is complete -- a matrix of
blocks (rows) vs operating modes (columns):

```
                 | Camera 4K | Video play | Gaming | Idle screen-on | Standby
-----------------|-----------|------------|--------|----------------|--------
CPU big cluster  |  450 mW   |   80 mW    | 900 mW |     15 mW      |  0 (PG)
CPU LITTLE       |  200 mW   |  120 mW    | 250 mW |     40 mW      |  2 mW
GPU              |  150 mW   |   60 mW    |1800 mW |      0 (PG)    |  0 (PG)
NPU              |  800 mW   |    0 (PG)  |  50 mW |      0 (PG)    |  0 (PG)
ISP              |  900 mW   |    0 (PG)  |   0    |      0 (PG)    |  0 (PG)
Video codec      |  300 mW   |  250 mW    |  80 mW |      0 (PG)    |  0 (PG)
Memory ctrl+DRAM |  700 mW   |  350 mW    | 950 mW |     90 mW      |  8 mW
Display          |  250 mW   |  240 mW    | 280 mW |    180 mW      |  0
Always-on island |   10 mW   |   10 mW    |  10 mW |     10 mW      |  3 mW
-----------------|-----------|------------|--------|----------------|--------
TOTAL            | ~3.8 W    |  ~1.1 W    | ~4.3 W |    ~0.34 W     | ~13 mW
```

Each cell = (block's C_eff and leakage) x (activity in that mode) x (V,f in that mode).
This matrix drives: battery-life projections, thermal design, PMIC rail planning,
per-block power BUDGETS handed to design teams, and the UPF power-state table.

```
Per-block budget flowdown (what a mid-level engineer owns):
  SoC budget 4.3W gaming -> GPU gets 1.8W -> shader array gets 1.2W
  -> your block gets 85 mW at 800 MHz, 0.75V, with an assumed 40% activity
  -> you sign off against it at every netlist drop, and explain every
     regression (new feature? activity assumption broke? clock gating lost?)
```

### 2.2 Where the Activity Numbers Come From (Before Silicon)

```
Quality ladder (worst to best):
1. Analytical guess: utilization assumptions from performance models
   (spreadsheet alpha per sub-block)                      error: 2x or worse
2. Architectural simulator traces: per-block event counts (cache accesses,
   ALU issues) x energy-per-event                          error: 30-50%
3. RTL simulation activity (SAIF/FSDB) on short directed tests   20-30%
4. RTL/gate activity from EMULATION on real software (boot, game frame,
   LLM inference layer)                                    10-20%
5. Gate-level + parasitics + real workload windows         5-15%
```

The jump from (3) to (4) matters: short testbenches systematically OVERESTIMATE
activity (they exercise the block continuously) and UNDERESTIMATE idle/clock-gated
residency. Real software has phases.

---

## 3. RTL Power Estimation -- The Shift-Left Toolchain

Waiting for gate-level signoff to discover a power bust is a schedule disaster, so the
industry runs power analysis from RTL onward. Current tool landscape (2025-2026):

| Tool | Vendor | Notes |
|------|--------|-------|
| PrimePower RTL | Synopsys | RTL power with implementation-calibrated models; feeds the same engine as gate-level PrimePower signoff |
| Joules (+ Joules RTL Design Studio) | Cadence | Time-based RTL power; integrates with Genus/Innovus models; guidance for gating fixes |
| PowerArtist | Keysight (acquired from Ansys, 2025 -- divested as a condition of the Synopsys-Ansys merger) | The classic RTL design-for-power platform: analysis + automatic reduction guidance |
| PowerPro | Siemens EDA | Sequential/observability-based gating analysis and insertion (SLEC-verified) |

```
What RTL power tools actually do:
1. Elaborate RTL, do a fast synthesis-like mapping to estimate gates/wires
2. Read activity (RTL sim FSDB/SAIF, or emulation activity database)
3. Report per-hierarchy: dynamic power, leakage, CLOCK power split,
   clock-gating efficiency, memory access power
4. Flag reduction opportunities:
   - FFs clocking with stable D (wasted clock) -> gating candidates
   - Memories read every cycle but data used rarely -> access gating
   - High-toggle nets feeding disabled logic -> operand isolation
Accuracy: typically within 10-20% of gate-level if activity is realistic --
good enough to TREND and to catch architecture-level regressions.
```

**Power regression in CI:** mature teams run RTL power on fixed workload snippets at
every RTL drop, with budgets per block. A merge that drops CGE from 78% to 60% gets
flagged like a failing test. Being able to describe this flow is a strong mid-level
interview signal.

---

## 4. Glitch Power -- The Activity You Didn't Mean to Have

A glitch is a spurious transition: a gate output toggles 2+ times in one cycle because
its inputs arrive at different times, before settling to the final value. The wasted
charge is real power.

```
       a ----\
              XOR ---- y        a and b both rise, but b arrives 80 ps late:
       b ----/                  y pulses high for 80 ps -> 2 extra transitions

Glitch generation: unequal arrival times at reconvergent logic
Glitch PROPAGATION: downstream gates re-toggle too (amplifies through
                    arithmetic: adder carry chains, multiplier arrays
                    are the worst -- deep reconvergent structures)
Inertial filtering: a pulse shorter than a gate's switching delay dies out;
                    at advanced nodes gates are FAST, so more glitches
                    survive and propagate further.
```

**Why it's a headline topic now:** at 7nm and below, glitch power is commonly **25-40%
of dynamic power** in datapath-heavy designs (GPU/video-class blocks reported worst).
It grew because logic got faster (less filtering), wires got relatively slower (more
arrival-time spread), and datapaths got wider.

**Measurement subtlety (interview checkpoint):**
```
Zero-delay RTL simulation CANNOT see glitches (all transitions instantaneous)
  -> RTL-based power UNDERESTIMATES datapath power systematically
Options:
  - Gate-level sim with SDF timing -> real glitches, but slow
  - Vectorless/statistical glitch estimation in power tools (propagate
    arrival-time windows; PrimePower/Joules support glitch modes)
  - Delay-annotated emulation hybrids
A "transparent" (full-swing) vs "filtered" (partial-swing) glitch split is
reported by signoff tools; filtered glitches still burn partial CV^2.
```

**Reduction techniques:** path balancing (equalize arrival times into reconvergent
logic), operand isolation, register retiming to cut deep combinational clouds,
gating-off of don't-care inputs, and at the architecture level: avoid letting wide
datapaths free-run on invalid data (the same valid-gating that helps clock power).

---

## 5. Emulation-Based Power -- Activity at Software Scale

Simulation gives you microseconds of activity; real power questions need *seconds* of
software: OS boot, a game frame, an LLM inference pass, a 5G call.

```
Flow (all vendors have an equivalent):
  1. Map RTL onto emulator (Cadence Palladium, Siemens Veloce, Synopsys ZeBu)
  2. Run REAL workload: billions of cycles
  3. Emulator streams a compact activity database (toggle counts per net
     per time window -- e.g., Palladium DPA, "Dynamic Power Analysis")
  4. Power tool (Joules / PrimePower / PowerArtist) converts windowed
     activity -> power-over-time waveform per block
  5. Find: peak windows (which 10 us window draws max power?), average per
     phase, idle residency, CGE on real software

What it catches that simulation never will:
  - The OS timer tick that keeps a "mostly idle" cluster at 30% clock activity
  - A driver bug that never lets the GPU enter its low-power state
  - DVFS governor thrash between operating points
  - The true peak window to feed dynamic IR drop signoff (vectors that
    actually happen, vs synthetic worst-case)
```

---

## 6. Silicon: On-Die Activity Monitoring and Power Telemetry

Once the chip exists, activity becomes something you MEASURE and CONTROL against.

### 6.1 The Monitoring Stack

```
Level 1: Performance/activity counters (per block)
  - issue counts, cache accesses, FLOP counts, busy/idle residency
  - free byproduct of the microarchitecture; coarse but cheap

Level 2: Digital power meters (DPM) / energy counters
  - Hardware computes a POWER PROXY each cycle: a weighted sum of selected
    activity signals, weights fit during characterization so the proxy
    tracks true power within a few %
      P_proxy = w0 + sum_i ( w_i * event_i )    (per block, per cycle)
  - Accumulated into energy counters firmware can read (e.g., the model
    behind Intel RAPL energy reporting on modern parts)
  - Fast (ns-us) and per-block -- this IS "block activity power" in silicon

Level 3: Analog telemetry
  - VRM/PMIC current sense (board-level truth, but ms-slow and per-rail)
  - On-die voltage droop monitors, thermal sensor grids (10s per die)

Level 4: Speed monitors
  - Ring oscillators / critical-path replicas: how fast IS this silicon
    at this V,T -- closes the AVS/AVFS loop
```

### 6.2 Closed-Loop Uses

```
Power capping:    firmware keeps a sliding-window average below a limit
                  (Intel RAPL power limits; NVIDIA power limit / nvidia-smi;
                  AMD PPT). Enforced by DVFS backoff.
Current limiting: protect the VRM and package: peak current (EDC) and
                  sustained current (TDC) limits trigger fast clock throttle.
Thermal control:  sensor grid -> throttle or migrate work before Tj_max.
Energy attribution: per-block energy counters let the OS/scheduler bill
                  energy to processes/VMs and let cloud operators meter it.
Turbo/boost:      the INVERSE use -- spend the headroom: if telemetry says
                  power/thermal/current budgets have slack, raise V/f
                  (all modern boost algorithms are telemetry-driven).
```

**The reaction-time hierarchy (memorize this -- it organizes every PM interview answer):**

| Timescale | Phenomenon | Mechanism |
|-----------|------------|-----------|
| ns | di/dt first droop | decap; droop detector + adaptive clock stretch |
| us | load steps, rail settling | regulator loop; fast DVFS (IVR/DLVR) |
| ms | workload phases | firmware DVFS governor, power capping |
| 100ms-s | thermal time constants | thermal throttling, fan/cooling control |
| s-min | job/rack level | scheduler placement, rack power smoothing |

### 6.3 Worked Example: Power Proxy Design

```
Task: build a power proxy for a matrix-multiply engine block.
Candidate events: mac_issue_count, operand_sram_rd, operand_sram_wr,
                  accum_writeback, clk_enabled_cycles

1. Run characterization workloads on silicon (or gate-level power on the
   same windows), recording true block power + event counts per window
2. Least-squares fit weights w_i  (per voltage/frequency point, or
   normalize events by f and fit in energy units)
3. Validate on held-out workloads: target within ~3-5% of measured power
4. Hardware: multiply-accumulate of ~5-10 counters x constants per window
   -- trivial area, runs continuously

Why not just measure current? Per-block attribution (one rail feeds many
blocks), and speed (proxy updates every window; VRM telemetry is ms-slow
and noisy). The proxy plus occasional analog calibration is the standard.
```

---

## 7. Component-Level Power Models -- Building P_block Bottom-Up

The `sum over nets` formula is conceptually right but never evaluated net-by-net at
the architecture or RTL level. Instead power is built from a handful of *component
primitives*, each with its own model and its own source for the three inputs (C, alpha,
leakage). This is exactly how McPAT/Wattch (architectural) and PrimePower/Joules (RTL)
decompose a block.

```
For each component:  P = P_dyn + P_leak,  P_dyn = E_op * (access rate)  OR  alpha*C*Vdd^2*f
                                          P_leak = N_dev * I_leak(Vt,V,T) * Vdd
Where each input comes from:
  C            <- parasitic extraction (RC) for gate-level; library C_load + wire-load
                  models for RTL; analytical area*cap-density for architectural
  E_op         <- characterized .lib energy tables (per-cell switching energy),
                  or CACTI/closed-form for arrays
  alpha (act.) <- SAIF/FSDB annotation (sim/emulation), or event counts from a perf
                  simulator (architectural), or assumed utilization (spreadsheet)
  I_leak       <- .lib leakage tables, per Vt flavor, scaled by state + temperature
```

| Component | Dynamic model | Leakage / static | Where the inputs come from |
|-----------|---------------|------------------|----------------------------|
| **Combinational logic** | `alpha*C_eff*Vdd^2*f` per net; C_eff = gate input caps + wire C; includes glitch CV^2 | sub-threshold + gate leakage per cell, Vt-mix dependent | C from extraction (.spef) or wire-load; alpha from annotation; E from .lib cell tables |
| **Flip-flop + clock tree** | per-FF: clock-pin CV^2 every cycle (alpha_clk=1 unless gated) + data-path CV^2 at alpha_data; **clock tree CV^2 dominates** | small | clock net C from CTS/extraction; gating efficiency (CGE) from activity |
| **SRAM / cache** | per-access read/write energy from **CACTI** (bitline + wordline + sense-amp + decoder + H-tree), times access rate | bitcell leakage (huge in large arrays) + **retention** power in drowsy/light-sleep | CACTI model (size, banks, ports, tech node); access rate from perf sim / SAIF |
| **Interconnect / wire** | `alpha*C_wire*Vdd^2*f`; repeaters add cell energy; long global wires dominate datapath C | repeater leakage | C_wire from extraction or per-mm cap density; alpha from net annotation |
| **I/O / PHY** | termination + driver energy per transition; often a fixed mW/Gbps per lane | bias currents (analog, ~always on) | datasheet / IP characterization; lane utilization |

**Clock-tree power is the headline.** The clock net toggles every cycle by definition
(`alpha_clk = 1.0`), it has the largest fanout and total capacitance on the die (CTS
buffers + all FF clock pins + the H-tree wires), so **clock power is commonly 30-40% of
total dynamic power** (higher in flop-dense, low-logic-depth designs). This is *why*
clock gating is the first-line dynamic-power lever (see
[Power_Reduction_Techniques](Power_Reduction_Techniques.md)): killing a cycle of clock
toggling on a gated FF removes the single biggest per-FF energy term.

**SRAM detail (CACTI's role).** [CACTI](https://en.wikipedia.org/wiki/CACTI) is the
standard analytical model for SRAM/cache access energy, delay, and area: given
(capacity, block size, associativity, #banks/ports, tech node) it returns per-read and
per-write energy by summing decoder, wordline, bitline (the dominant term -- precharge
+ discharge of long bitlines), sense-amp, and H-tree distribution energy. McPAT calls
CACTI for every array (regfile, caches, TLBs, buffers). For arrays, leakage and (in
drowsy/retention modes) *retention* power often exceed dynamic in idle-heavy blocks --
the reason large LLCs are aggressively power-gated way down or retained at low Vdd.

---

## 8. The Power-Modeling Abstraction Ladder

Power modeling has the same speed<->accuracy ladder as the performance modeling in
[Performance_Modeling_and_DSE](../01_Architecture_and_PPA/Performance_Modeling_and_DSE.md#1-the-modeling-fidelity-ladder)
-- and you move *down* it as the design firms up, trading runtime for fidelity. Each
rung needs both a structural model (what's instantiated) and an activity source.

| Level | Tool / example | Fidelity | Speed | Activity source |
|-------|----------------|----------|-------|-----------------|
| **Architectural** | **McPAT**, **Wattch** (+ **CACTI** for arrays, Orion/DSENT for NoC) | ±20-30% (un-calibrated worse) | instant (per-design-point) | event counts from a perf simulator (gem5/Sniper): cache accesses, ALU issues, regfile reads |
| **RTL power** | PrimePower RTL, Cadence **Joules**, **PowerArtist** (Keysight), PowerPro (Siemens) | ±10-20% vs gates (good activity) | minutes-hours | RTL sim SAIF/FSDB or emulation activity DB |
| **Gate-level** | **PrimePower** / PrimeTime-PX (Synopsys), Cadence Voltus/Joules gate mode | ±5-10% (signoff) | hours-days | SDF-annotated gate sim VCD/SAIF (captures glitch) |
| **SPICE / circuit** | HSPICE, Spectre, FineSim | golden (per-cell) | impractical above small blocks | actual transient waveforms |

```
Architectural (McPAT-style) = sum over components ( events_c * E_op,c ) + leakage
   - bottom-up: each component's E_op from CACTI/closed-form, area-derived caps
   - activity = performance-counter / simulator event counts (NOT real toggles)
   - used in early DSE: power as a first-class axis alongside perf and area
RTL power = fast-map RTL to gates, annotate real toggles -> per-hierarchy power
Gate PrimePower = mapped netlist + .spef parasitics + .lib energy + SDF activity
   - this is signoff; the only rung that sees real glitch power (Section 4)
SPICE = ground truth, used to CHARACTERIZE the .lib tables the upper rungs trust
```

**The calibration chain:** SPICE characterizes the `.lib` energy/leakage tables; those
feed gate-level PrimePower; gate-level results calibrate the RTL-power models; RTL/silicon
results calibrate the architectural McPAT coefficients. Each rung is only as good as the
rung below that anchored it -- an un-calibrated McPAT run can be 2x off, the same trap as
an un-validated cycle-accurate performance model.

---

## 9. Dynamic Power Management (DPM) -- Modeling Idleness as a State Machine

Sections 1-8 model power *while a block works*. But **real workloads are idle-dominated**
-- a CPU, GPU, or NPU spends most wall-clock time waiting (between frames, between
requests, between training steps). A peak-power or even average-active model says nothing
about the energy of those idle gaps, and *that energy is often the majority of the total*.
DPM is the discipline of spending idleness: detect idle, transition to a low-power state,
wake up in time. The canonical framework is Benini, Bogliolo & De Micheli's *Survey of
Design Techniques for System-Level Dynamic Power Management* (IEEE TVLSI, 2000).

### 9.1 The Power State Machine (PSM)

A power-manageable component is modeled as a finite-state machine. Each **state** has a
power level; each **transition** has an energy and a latency cost (you pay to go to sleep
and, more expensively, to wake up).

```
Power State Machine (generic):
                 wake (T_wake, E_wake)
   +--------+  <----------------------  +--------+
   | ACTIVE |                           | SLEEP  |   states carry: P_state
   | P_on   |  ---------------------->  | P_off  |   edges carry:  T_trans, E_trans
   +--------+   sleep (T_sl, E_sl)      +--------+

Real components have MANY inactive states (idle/clock-gated, retention, deep-sleep, off):
  deeper state -> lower P_state  BUT  longer T_wake and larger E_wake.

Example -- StrongARM SA-1100 (the survey's canonical PSM):
  RUN   ~400 mW   |  IDLE ~50 mW (fast exit)  |  SLEEP ~0.16 mW (slow, costly wake-up)
  Idle is cheap to enter/exit; Sleep saves ~3000x power but its wake cost is large.
  In the survey's two-state (On/Off) reduction, the SA-1100 Sleep state has a
  break-even time of ~160 ms -- only idle gaps longer than that pay off (Section 9.2).
```

### 9.2 Break-Even Time -- When Is Sleeping Worth It?

The core decision metric. The **break-even time** `T_be` is the minimum idle duration for
which entering a low-power state actually saves energy, after paying the transition cost.
If an idle period is shorter than `T_be`, sleeping *costs* more than it saves (you burn
wake-up energy for nothing) -- you must stay awake.

```
Two-state derivation (ACTIVE P_on <-> SLEEP P_off, transition time T_tr, transition power P_tr):

  Energy if you STAY AWAKE for idle time T_idle:     E_stay  = P_on  * T_idle
  Energy if you SLEEP (enter+exit cost + residency):  E_sleep = P_tr*T_tr + P_off*(T_idle - T_tr)

  Sleeping wins when E_sleep < E_stay. Setting equal and solving for the idle time:

           ___________________________________
          |                                    |
          |   T_be = T_tr  +  T_tr * (P_tr - P_on)          ... if P_tr > P_on (overhead)
          |                    -----------------                                              |
          |                       (P_on - P_off)                                              |
          |___________________________________|

  - When transition power P_tr <= P_on (e.g. SA-1100, where wake power ~= run power),
    T_be reduces to just T_tr -- the latency itself is the whole barrier.
  - When there's extra wake energy (mechanical inertia: disks; or large rush current),
    the second term adds the time needed to amortize that excess.
  - DEEPER states have larger T_tr/E_wake => larger T_be => need LONGER idle gaps to pay off.
```

Rationale, stated plainly: DPM **trades wake-up latency (a performance/responsiveness
cost) against leakage + idle-clock savings**. The break-even time quantifies that trade.
The whole game is predicting whether the *upcoming* idle period exceeds `T_be`.

### 9.3 The Three Policy Classes (Governors)

You don't know the future idle length, so a **policy** must guess. The survey's taxonomy:

| Class | Mechanism | Pro | Con |
|-------|-----------|-----|-----|
| **Timeout** | Sleep after the block has been idle `T_timeout` (often `T_timeout = T_be`) | simple, workload-agnostic, *safe* (tune by raising timeout) | wastes the timeout window of power every idle period; trades efficiency for safety. Karlin's result: timeout = T_be is **2-competitive** (<=2x ideal energy) |
| **Predictive** | Predict the idle length from history (e.g. short busy => long idle), sleep *immediately* if predicted idle > T_be | no wasted timeout window; can wake *before* the request (hide T_wake) | mispredicts: **over-prediction** = performance penalty (woke late), **under-prediction** = wasted power. Quality = safety vs efficiency |
| **Stochastic (MDP)** | Model workload + PSM as a Markov Decision Process; solve for the policy minimizing expected energy under a performance constraint | provably optimal for the modeled distribution; handles multi-state and constraints natively | needs a workload model; optimum only as good as the model; non-stationary workloads need adaptation |

```
Energy accounting for ANY policy over a run:
  E_total = sum over states ( residency_s * P_state,s )  +  sum over transitions ( N_trans * E_trans )
            \________________ time IN states ________/      \____ cost of MOVING between states ____/
  A good policy maximizes residency in deep states for idle gaps > T_be while keeping the
  transition-energy term (and the latency it implies) small.
```

### 9.4 CPU Instantiation -- C-states, P-states, and the PCU

The CPU is the most-engineered DPM instance, split into two orthogonal axes:

```
ACPI C-states = the IDLE PSM (DPM proper -- "how asleep when not running")
  C0 = active (executing).  C1 = halt (clock-gated core).  C3/C6/C7... = deeper:
       flush caches, power-gate the core, save state -> lower P_off but larger T_wake.
  Deeper C-state <-> larger break-even time: the OS/firmware idle governor (e.g. Linux
  'menu'/'TEO' cpuidle) PREDICTS the idle duration and picks the deepest C-state whose
  T_be fits -- a literal predictive DPM governor (Section 9.3).

ACPI P-states = DVFS while ACTIVE (voltage/frequency operating points, NOT idle)
  P0 highest V/f ... Pn lowest. Reduces alpha*C*V^2*f when running but slower.
  (DVFS is detailed in Power_Reduction_Techniques; counters drive P-state choice -- see Q8.)

Package C-states (PC2/PC6...) = the WHOLE-PACKAGE PSM: once ALL cores are in a deep
  core C-state, the UNCORE (LLC, ring/mesh, memory controller, PLLs) can also retire to
  a package-level low-power state. Bigger savings, bigger wake cost -- a higher-T_be PSM.

The PCU (Power Control Unit) / PMU = the on-die microcontroller running these governors:
  reads telemetry + counters (Section 6.2), enforces C-state/P-state transitions, power
  caps (RAPL), and turbo. It IS the hardware power manager of the PSM.
```

This makes Section 6's telemetry and Section 9's PSM one loop: the PCU *measures* activity
(proxies/counters) and *acts* on the PSM (C/P-states) -- estimate -> monitor -> manage.

---

## 10. Counter-Based Power Proxies -- Runtime / Online Power

Online power management (Section 9) needs power *now*, per block, far faster and cheaper
than analog current sense. The answer is a **power proxy**: a weighted sum of activity
counters whose weights are fit (offline) so the proxy tracks true power. This generalizes
the digital power meter of Section 6.2 and is the standard industrial mechanism.

```
P_proxy = w0 + sum_i ( w_i * event_i )      events = issue counts, cache/SRAM accesses,
                                            FP-width usage, clk-enabled cycles, ...
Weights w_i fit by least-squares against measured/gate-level power per (V,f) point;
accumulated into an ENERGY counter firmware reads. Accuracy ~3-5% on held-out workloads.
```

| Mechanism | What it is | Notes |
|-----------|-----------|-------|
| **Intel RAPL** (Running Average Power Limit) | Per-domain energy counters + power-limit registers, exposed via MSRs, updated ~1 ms | Domains: **PKG** (whole socket), **PP0** (cores), **PP1** (iGPU, client parts), **DRAM** (server). Early parts used a counter-based *model*; later parts blend on-die sensing. Drives power *capping* (PL1 sustained / PL2 burst), enforced by the PCU via DVFS backoff |
| **IBM POWER on-die "power proxy"** | Per-core hardware estimator: weighted sum of activity events accumulated continuously | Feeds the on-chip power-management controller (OCC) for per-core DVFS, capping, and idle management -- a textbook silicon power proxy |
| **Performance-counter regression models** | Software/OS builds a linear (or ML) model from existing PMU counters (IPC, LLC misses, etc.) to estimate power without dedicated HW | Used where no energy counter exists; the per-process/per-VM **energy attribution** and cloud metering mechanism |

The deliberate trade: a proxy gives **per-block attribution** (one VRM rail feeds many
blocks, so analog sense can't separate them) and **speed** (per-window vs ms-slow, noisy
rail telemetry). Standard practice pairs a fast proxy with occasional analog calibration.

---

## 11. Per-Architecture Block Power -- CPU, GPU, NPU

The same `P = alpha*C*V^2*f + leak` and PSM machinery, instantiated per architecture.
The art is mapping *occupancy/utilization* (the architecture team's number) to per-block
alpha (the power team's number) -- the bridge this whole page is about.

```
CPU  (P_core + P_caches + P_uncore)
  core    : fetch/decode/rename/issue/ALU/FPU/LSU -- per-unit E_op * access counts
            (McPAT decomposition); clock tree 30-40% (Section 7)
  caches  : L1/L2 per-access (CACTI) + LLC leakage/retention (often power-gated down)
  uncore  : ring/mesh NoC, memory controller, PLLs -- significant FIXED + idle floor;
            only retired in package C-states (Section 9.4)
  activity from: perf counters (issue/access rates) -> proxy (Section 10)

GPU  (SM/CU array + register file + shared mem + L2 + HBM)
  SM array      : thousands of lanes; power scales with OCCUPANCY (active warps/SM) ->
                  alpha. Low occupancy = idle lanes = wasted leakage + clock
  register file : huge, multi-ported SRAM -- a top energy consumer (CACTI-modeled);
                  read/write per instruction operand
  shared mem/L2 : per-access energy; bank-conflict stalls lower effective activity
  HBM           : pJ/bit * bytes moved -- bandwidth-bound phases are HBM-power-bound
  capping       : NVIDIA power limit / nvidia-smi enforce a board cap via DVFS (Section 6.2)

NPU  (systolic/MAC array + vector unit + on-chip SRAM + HBM + ICI)
  systolic array: alpha ~ MAC-array utilization for the operator (a 256x256 array at 40%
                  mapping efficiency burns clock+leakage on 60% idle PEs)
  vector/SFU    : activation/normalization -- bursty, lower duty cycle
  SRAM (scratch): weight/activation buffers -- CACTI per-access + retention
  HBM + ICI     : off-chip + inter-chip (NVLink/ICI) energy per byte; collective/comm
                  phases are interconnect-power-bound, compute idle
  POWER IS OPERATOR-LEVEL: each layer (GEMM vs attention vs elementwise) has a distinct
  block-activity signature -- exactly what an operator-level perf+power model predicts.
  Cross-link: the NeuSim operator-level example in
  [Performance_Modeling_and_DSE](../01_Architecture_and_PPA/Performance_Modeling_and_DSE.md)
  produces the per-operator activity these NPU block-power numbers consume.
```

**NPU power gating is a fine-grained DPM instance.** Because an NPU's array is idle
during memory-bound or communication phases, accelerators power-gate (or clock-gate)
unused PE tiles, vector units, and SRAM banks at sub-operator granularity. This is the
PSM of Section 9 applied per-tile: each tile has active/clock-gated/power-gated states
with their own break-even times, and the dataflow schedule effectively *is* the DPM
policy -- it decides which tiles sleep when, and for how long (must exceed the tile's
`T_be`). DVFS across the array adds the P-state axis.

---

## 12. Peak Power, Power Virus, and TDP -- Block Activity at Its Worst

```
Hierarchy of power numbers for one chip (illustrative ratios):
  Idle                                ~0.05-0.15x TDP
  Typical application                 ~0.4-0.7x TDP
  TDP (thermal design power)           1.0x  -- sustained cooling design point
  Electrical design max (Pmax/EDP)     1.3-2.5x TDP -- worst realistic burst
  Power virus ceiling                  up to ~2-3x TDP -- adversarial

Power virus: a synthetic workload that maximizes alpha everywhere at once --
all lanes computing, all caches missing, maximum toggle data patterns.
Used ON PURPOSE for: dynamic IR signoff vectors, VRM/package stress test,
throttling-mechanism validation. Guarded against in production by EDC/TDC
current limits and fast throttle (chips are NOT built to sustain it).

Why TDP < Pmax is fine: thermal time constants are long (the heatsink
averages over seconds), but the POWER DELIVERY must survive the burst --
hence different limits enforced at different timescales (PL1/PL2-style
sustained vs burst limits, peak current throttle at us scale).
```

**AI-cluster extension:** when thousands of accelerators synchronize (training), the
burst/idle pattern that power limits handle per-chip becomes a megawatt square wave at
the facility. See the DVFS section of
[Power_Reduction_Techniques](Power_Reduction_Techniques.md) (GB300 NVL72 power
smoothing: per-GPU energy storage, ramp-limited caps, controlled burn-down) and the
AI-infra notes on rack-scale design -- interviewers for AI-hardware roles love this
bridge.

---

## 13. Interview Q&A

### Q1: "What does 'block activity power' mean to you?"

**A:** The activity-dependent component of a block's power: P = alpha*C*V^2*f summed
over the block, where alpha is set by utilization. Concretely it shows up three ways:
at design time as the per-block/per-mode power matrix built from predicted activity; in
verification as measured activity (SAIF/emulation) annotated onto the netlist; in
silicon as activity-counter-based power proxies that firmware uses for capping, boost,
and attribution. The unifying skill is connecting utilization numbers to watts.

### Q2: "Your block's measured silicon power is 30% above the signoff estimate. Debug path?"

**A:** Bisect model vs reality. (1) Check leakage first: measure at idle with clocks
gated -- if idle power is high, it's leakage (corner/temperature/Vt-mix issue), not
activity. (2) If dynamic: compare assumed vs real activity -- read activity counters
during the failing workload; the most common cause is an activity assumption (e.g.,
40% utilization assumed, 75% real, or clock gating not engaging due to a software/
driver setting). (3) Check voltage: is AVS delivering the assumed voltage or running
the guardbanded default? (4) Check glitch: if signoff used zero-delay RTL activity,
datapath glitch power was underestimated -- compare against SDF gate-level on a window.
(5) Only then suspect extraction/library model errors.

### Q3: "Why can't RTL simulation activity be trusted for peak-power signoff?"

**A:** Three gaps: testbenches don't represent real software phases (overestimate
sustained activity, underestimate true synchronized peaks); zero-delay RTL sim can't
produce glitches (underestimates datapath power 25-40% at advanced nodes); and short
sim windows miss the worst-case alignment of events (the real peak 10 us window comes
from emulation of actual workloads). Signoff practice: emulation-derived windows for
realistic peaks + synthetic power-virus vectors for the electrical worst case.

### Q4: "Design a mechanism for the OS to know each core's power consumption in real time."

**A:** Per-core digital power meter: select 5-10 microarchitectural events that
correlate with power (instructions issued by class, cache accesses, FP width usage,
clock-enabled cycles), compute a weighted sum per window in hardware, weights
characterized against measured power per V/f point, accumulate into an energy counter
(this is the RAPL-style model). Expose via MSR/registers. Calibrate periodically
against rail telemetry. Discuss error sources: workload-dependent residual (~3-5%),
temperature effect on leakage (add T-sensor term), and aliasing if the window is too
long relative to workload phases.

### Q5: "What is a power virus and why do designers care?"

**A:** An adversarial workload maximizing simultaneous switching -- the realistic
ceiling of block activity. Designers care because it sizes the power delivery (VRM,
package current, grid IR at the di/dt event when it starts), validates throttle
mechanisms, and bounds the gap between TDP (thermal, sustained) and Pmax (electrical,
instantaneous). Production chips survive it via fast current-limit throttling, not by
provisioning cooling for it.

### Q6: "How would you reduce glitch power in a wide multiplier without touching frequency?"

**A:** Balance arrival times into reconvergent stages (synthesis path-balancing /
delay-equalization options), register-retime to shorten the deepest combinational
cloud, isolate operands so invalid cycles don't propagate garbage, consider
sign-magnitude or zero-byte-aware encodings if the data is biased, and ensure the
multiplier's inputs are clock-gated/held stable when results are unused -- glitches
scale with input arrival spread AND with how often inputs change uselessly. Verify with
SDF-annotated gate-level power on the same activity window before/after.

### Q7: "Average power meets budget but the chip still resets in the field. Where do you look?"

**A:** Peak phenomena that averages hide: di/dt droop on synchronized events (check
droop monitor logs / repeat with adaptive clocking disabled in a lab build), rush
current on a power-domain wake collapsing a shared rail, current-limit (EDC) throttle
misconfiguration, or a workload-correlated thermal hotspot (sensor grid too sparse to
catch it). The instinct "power problem = average power" is exactly what this question
screens out; the reaction-time hierarchy (ns droop / us rail / ms DVFS / s thermal)
gives the structured answer.

### Q8: "How do activity counters enable better DVFS than utilization alone?"

**A:** Utilization says busy/idle; counters say WHAT KIND of busy. A memory-bound phase
shows high stall counts and low issue counts -- frequency can drop with little
performance loss (performance is bandwidth-limited), saving V^2*f power. A
compute-bound phase shows the opposite, wanting max frequency. Modern governors
(hardware P-state / firmware) classify phases from counters and pick operating points;
the same logic at cluster scale drops accelerator clocks during communication phases of
training, or during memory-bound decode in LLM inference serving.

### Q9: "A block is idle. Should it sleep? Walk me through the decision."

**A:** Compute the break-even time `T_be` from its power state machine: for a two-state
model `T_be = T_tr + T_tr*(P_tr - P_on)/(P_on - P_off)`, which reduces to just the wake
latency `T_tr` when wake power isn't inflated. Sleep only if the *expected* idle period
exceeds `T_be` -- below it you burn more wake-up energy than you save. Since you don't
know the idle length, a policy predicts it: a timeout (sleep after idle >= T_be; safe,
2-competitive per Karlin, but wastes the timeout window), a predictive governor (guess
from history, sleep immediately, risk over/under-prediction), or a stochastic MDP policy
(optimal for a modeled workload). Deeper states have larger T_be, so they need longer
idle gaps -- exactly what the Linux cpuidle menu/TEO governor does picking C-states.

### Q10: "Why are peak-power models useless for battery/energy, and what replaces them?"

**A:** Real workloads are idle-dominated -- most wall-clock time is spent between frames,
requests, or training steps, so total energy is dominated by idle residency and
transition costs, not the active peak. A peak (or even average-active) model ignores the
majority of the energy. The replacement is DPM modeling: a power state machine per block,
energy = `sum(residency_s * P_state,s) + sum(N_trans * E_trans)`, with a governor that
maximizes deep-state residency for idle gaps above break-even. On a CPU this is C-states +
package C-states driven by the PCU; on an NPU it's per-tile clock/power gating scheduled
by the dataflow. You size cooling to peak, but you size *battery life* to this integral.

---

## 14. Numbers to Remember

| Quantity | Value |
|----------|-------|
| Datapath alpha, real data | 0.05-0.25 |
| Clock net alpha | 1.0 (by definition) |
| Glitch share of dynamic power, ≤7nm datapath designs | 25-40% (up to ~60% reported in GPU-class blocks) |
| RTL power accuracy vs gate-level (good activity) | 10-20% |
| Emulation activity scale | billions of cycles (vs ~10^5-10^6 for sim) |
| Power proxy accuracy (fitted, per block) | ~3-5% |
| Clock-tree share of dynamic power | ~30-40% (higher in flop-dense blocks) |
| McPAT / architectural power accuracy (calibrated) | ~20-30% (worse un-calibrated) |
| Break-even time (2-state, P_tr<=P_on) | T_be = T_tr (the wake latency itself) |
| Timeout=T_be competitiveness (Karlin) | <=2x ideal energy (2-competitive) |
| SA-1100 PSM (survey canonical) | Run ~400 mW / Idle ~50 mW / Sleep ~0.16 mW; Sleep T_be ~160 ms |
| RAPL domains | PKG (socket) / PP0 (cores) / PP1 (iGPU) / DRAM; ~1 ms update |
| Pmax vs TDP | ~1.3-2.5x (power virus up to ~3x) |
| Reaction-time hierarchy | ns: clock stretch / us: regulator / ms: firmware DVFS / s: thermal |
| GB300 NVL72 power smoothing | ~65 J storage per GPU; up to ~30% peak grid demand reduction |

---

*Mid-level interview focus: connect utilization to watts at every stage -- predicted
(power matrix, McPAT/CACTI bottom-up), measured (SAIF/emulation), live
(counters/proxies/RAPL), and MANAGED (PSM / break-even / C-P-states / DPM governors).
Cross-reference: [Power_Fundamentals](Power_Fundamentals.md),
[Power_Reduction_Techniques](Power_Reduction_Techniques.md),
[Power_Analysis_and_Signoff](Power_Analysis_and_Signoff.md),
[Performance_Modeling_and_DSE](../01_Architecture_and_PPA/Performance_Modeling_and_DSE.md).*
