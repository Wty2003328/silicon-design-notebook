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

## 7. Peak Power, Power Virus, and TDP -- Block Activity at Its Worst

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

## 8. Interview Q&A

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

---

## 9. Numbers to Remember

| Quantity | Value |
|----------|-------|
| Datapath alpha, real data | 0.05-0.25 |
| Clock net alpha | 1.0 (by definition) |
| Glitch share of dynamic power, ≤7nm datapath designs | 25-40% (up to ~60% reported in GPU-class blocks) |
| RTL power accuracy vs gate-level (good activity) | 10-20% |
| Emulation activity scale | billions of cycles (vs ~10^5-10^6 for sim) |
| Power proxy accuracy (fitted, per block) | ~3-5% |
| Pmax vs TDP | ~1.3-2.5x (power virus up to ~3x) |
| Reaction-time hierarchy | ns: clock stretch / us: regulator / ms: firmware DVFS / s: thermal |
| GB300 NVL72 power smoothing | ~65 J storage per GPU; up to ~30% peak grid demand reduction |

---

*Mid-level interview focus: connect utilization to watts at every stage -- predicted
(power matrix), measured (SAIF/emulation), and live (counters/proxies/capping).
Cross-reference: Power_Fundamentals.md, Power_Reduction_Techniques.md,
Power_Analysis_and_Signoff.md.*
