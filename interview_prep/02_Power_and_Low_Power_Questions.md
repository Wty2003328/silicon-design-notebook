# 02 — Power and Low Power: Interview Questions

Consolidated interview Q&A and worked problems from every page in `02_Power_and_Low_Power/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## Block Activity Power -- Estimation, Monitoring, and Management

*From [Block_Activity_and_Power.md](../02_Power_and_Low_Power/02_Block_Activity_and_Power.md)*

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

## Low Power Design -- The Complete Interview Bible

*From [Power_Reduction_Techniques.md](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md)*

### Q1: Derive P = alpha * C * V^2 * f from first principles.

**A:** Consider a node with load capacitance C. During a 0->1 transition, charge Q = CV flows from VDD through the PMOS to charge C. Energy from VDD = Q * VDD = CV^2. Of this, (1/2)CV^2 is stored in C, and (1/2)CV^2 is dissipated in the PMOS channel. During 1->0, the stored (1/2)CV^2 is dissipated in the NMOS. Total energy per full cycle = CV^2, all dissipated as heat. With switching activity alpha (probability of 0->1 per clock) and clock frequency f, power = alpha * C * V^2 * f. This is exact regardless of transistor sizes (energy loss is intrinsic to charging a capacitor through a resistor).

### Q2: Why does leakage increase exponentially with temperature? Quantify it.

**A:** Sub-threshold leakage is I_sub = I_0 * exp(-Vth / (n * kT/q)). Temperature appears in two places: (1) the thermal voltage Vt = kT/q increases linearly with T, making the exponent less negative; (2) Vth decreases with T at about -1 to -2 mV/K, further reducing the exponent magnitude. Both effects increase leakage exponentially. Rule of thumb: leakage roughly doubles per 10C rise. From 25C to 125C (100C delta), leakage increases by 2^10 ~ 1000x. In practice, the increase is somewhat less (~100-300x) due to mobility reduction at high temperatures partially compensating, but it remains a dominant concern.

### Q3: Explain the difference between clock gating and power gating. When do you use each?

**A:** Clock gating stops the clock to idle blocks, eliminating switching power (dynamic) but leaving supply connected (leakage unchanged). It's lightweight: single ICG cell, ~1 cycle enable latency, no isolation or retention needed. Use for blocks that idle frequently for short periods (tens of cycles). Power gating disconnects VDD entirely, eliminating BOTH dynamic and leakage power. It requires header/footer switches, isolation cells, retention registers, and a power-on sequence (microseconds of latency). Use for blocks that idle for long periods (milliseconds+) where leakage savings justify the overhead. In a mobile SoC, both are used: clock gating for fine-grain (register-level) and power gating for coarse-grain (block-level).

### Q4: Size a header switch for a 2mm^2 domain with 200mA peak current.

**A:** Target IR drop budget: 5% of VDD = 5% * 0.8V = 40mV. Required switch resistance: R = V/I = 40mV / 200mA = 0.2 ohm. For a PMOS header in 7nm with R_per_um ~ 200 ohm*um: total width = 200/0.2 = 1000 um. With standard header cells of 10um width each: need 100 header cells. Distribute them uniformly along the VDD rail across the 2mm^2 domain. For rush current: C_domain ~ 20nF (10nF/mm^2), with 10us ramp time: I_rush = 20nF * 0.8V / 10us = 1.6mA (easily handled by 100 header cells). Use daisy-chain turn-on with 1us delay between groups of 10 cells.

### Q5: Explain sub-threshold slope and the Boltzmann tyranny. Why does it matter for low-power design?

**A:** Sub-threshold slope (SS) is the gate voltage change needed to reduce drain current by 10x: SS = n*kT/q*ln(10) = 60mV/decade minimum at room temperature (when n=1). This means to maintain a 10^6 Ion/Ioff ratio, we need Vth > 6 * 60mV = 360mV. This sets a floor on VDD (need VDD > Vth + overdrive for adequate speed). Since P ~ V^2, this floor limits how low we can scale power. At 7nm, SS is typically 65-70 mV/decade. Novel devices like tunnel FETs (TFETs) and negative capacitance FETs (NCFETs) aim to break this 60mV/decade limit to enable lower VDD operation.

### Q6: What is GIDL and when does it matter?

**A:** GIDL (Gate-Induced Drain Leakage) occurs when the gate is at 0V and drain is at VDD, creating a strong electric field at the gate-drain overlap region. This field causes band-to-band tunneling -- electrons tunnel from the valence band to the conduction band, generating electron-hole pairs. GIDL matters in: (1) DRAM, where it discharges the storage capacitor (reduces retention time); (2) FinFET devices with thin gate oxide and wrap-around gate (larger overlap area); (3) power-gated domains where header switches have high Vds when the domain is off. GIDL is typically 5-10% of total leakage but can be significant at low temperatures (where sub-threshold leakage drops but GIDL stays roughly constant).

### Q7: Describe the complete power-up sequence for a power-gated domain. Why does ordering matter?

**A:** The sequence is: (1) Deassert SLEEP (enable header switches, daisy-chain for rush current control); (2) Wait for virtual VDD to stabilize (monitor ramp or use fixed delay); (3) Assert RESET to all FFs in the domain; (4) Deassert RESET synchronously (to avoid recovery/removal violations); (5) Deassert ISOLATION (outputs now driven by live logic); (6) Assert RESTORE (retention FFs recover saved state); (7) Deassert RESTORE; (8) Resume operation. Ordering is critical: if ISOLATION is deasserted before power is stable, the domain outputs are floating/garbage and corrupt the always-on domain. If RESTORE happens before RESET, the reset overwrites the restored values. If RESET is deasserted asynchronously, FFs may violate recovery/removal timing.

### Q8: What is UPF and how does it differ from RTL?

**A:** UPF (Unified Power Format, IEEE 1801) is a separate TCL-based specification that defines power intent: power domains, supply networks, power states, isolation, retention, level shifters, and power switches. RTL describes function; UPF describes how power is managed. RTL cannot express concepts like "this block can be powered off" or "these outputs need isolation when the domain is off." UPF is read by synthesis tools (to insert special cells), simulation tools (to model power-down behavior), and P&R tools (to physically implement the power architecture). Without UPF, the tools would not know which cells need always-on supply, where to place header switches, or which outputs to isolate.

### Q9: Compare leakage reduction techniques: multi-Vt, power gating, and body biasing.

**A:** Multi-Vt: assigns HVT to non-critical paths (5-10x leakage reduction per cell, no area overhead, works always, standard flow). Power gating: turns off entire domains (nearly 100% leakage elimination when off, but requires isolation/retention/switches, only works for long idle periods). Body biasing: applies reverse body bias to increase Vth in idle mode (5-20x reduction, requires bias generators and distribution network, less effective in FinFET). Best practice: use multi-Vt always (baseline), add power gating for large blocks with long idle periods, use body biasing for fine-grain control in planar CMOS. In modern FinFET designs, multi-Vt + power gating is the dominant combination.

### Q10: How do you estimate the break-even time for power gating?

**A:** Power gating has overhead: energy to power-down (save state, ramp switch), energy to power-up (ramp, restore, re-initialize), and always-on leakage of retention FFs and isolation cells. Break-even time = overhead energy / leakage power saved. Example: if power-up/down costs 10nJ and leakage saved is 5mW, break-even = 10nJ / 5mW = 2us. If the block idles for less than 2us, power gating wastes energy. In practice, add margin: only gate if expected idle > 3-5x break-even. Software/firmware must predict idle duration, which is why DVFS (lower latency) is preferred for short idle periods and power gating for long idle periods.

### Q11: Explain IR drop and its impact on timing. How is it analyzed?

**A:** IR drop is voltage reduction across the power distribution network (PDN) due to resistive losses. If VDD at a cell drops from 0.80V to 0.76V (50mV drop), the cell's delay increases because drive current is proportional to (VDD-Vth)^alpha. A 50mV drop at 0.8V could increase delay by 5-10%. Static IR drop uses average current; dynamic IR drop captures transient surges (e.g., at clock edges when thousands of FFs switch simultaneously). Dynamic IR drop can be 2-3x worse than static. Analysis tools (Voltus, RedHawk) take switching activity + PDN model and compute per-instance voltage, which is fed back to STA for timing derating. Worst-case dynamic IR drop determines if the PDN needs more straps, wider metals, or additional decoupling capacitance.

### Q12: What is the relationship between activity factor and different types of signals?

**A:** Clock signals: alpha = 1.0 (toggle every cycle by definition). Data signals: alpha = 0.05-0.3 (depends on workload, randomness). Address buses: alpha ~ 0.2-0.5 (depends on access pattern). Reset signals: alpha ~ 0 (toggle only at reset). For power estimation, clock network alpha=1.0 makes it the dominant dynamic power consumer (30-50% of total). This is why clock gating is the single most effective dynamic power reduction technique. Accurate alpha values require simulation with representative workloads; using default alpha=0.5 can overestimate power by 2-3x.

### Q13: How do high-k dielectrics reduce gate leakage? Explain with the EOT concept.

**A:** Gate leakage (tunneling) depends exponentially on oxide thickness: thinner oxide = exponentially more tunneling. But thinner oxide gives higher gate capacitance = better transistor control. High-k dielectrics (like HfO2, k~25 vs SiO2 k~3.9) provide the same capacitance with a physically thicker layer: C = epsilon*k/T. EOT (Equivalent Oxide Thickness) = T_physical * (3.9/k_highk). Example: 5nm HfO2 has EOT = 5*(3.9/25) = 0.78nm. This gives the capacitance of a 0.78nm SiO2 gate but the tunneling barrier of a 5nm film, reducing gate leakage by 100-1000x. High-k was introduced at the 45nm node (Intel 2007) and enabled continued gate length scaling.

### Q14: What is thermal runaway and how do you prevent it in a design?

**A:** Thermal runaway is a positive feedback loop: leakage generates heat, heat increases temperature, higher temperature increases leakage exponentially. If the cooling system cannot remove heat fast enough, temperature rises until the chip fails. Prevention: (1) Design the PDN and package for adequate thermal dissipation (R_thermal * P_max < T_max - T_ambient); (2) Use on-chip temperature sensors (one per major block) feeding a Dynamic Thermal Management (DTM) controller; (3) DTM reduces frequency/voltage when temperature exceeds thresholds; (4) Hardware thermal trip circuit forces emergency shutdown above critical temperature (typically 125C junction); (5) At design time, perform thermal simulations to verify the power budget at worst-case workload doesn't exceed cooling capacity.

### Q15: Explain operand isolation with a concrete example showing power savings.

**A:** Consider a 32-bit multiplier used only when `valid` is high (25% of cycles). Without operand isolation, random input toggling causes the multiplier's internal nodes to switch uselessly 75% of the time. With operand isolation, the inputs are AND-gated with `valid`:

```verilog
wire [31:0] a_gated = valid ? a : 32'd0;
wire [31:0] b_gated = valid ? b : 32'd0;
wire [63:0] product = a_gated * b_gated;
```

When valid=0, inputs are 0, and the multiplier internals don't toggle (0*0 = 0, no transitions). Power savings: the multiplier has ~10,000 internal nodes with average alpha=0.2. Without isolation: P_mult = 0.2 * C * V^2 * f * 100% of time. With isolation: P_mult = 0.2 * C * V^2 * f * 25% + P_mux_overhead. Savings ~ 75% of multiplier dynamic power minus the small MUX (AND gate) overhead. The 64 AND gates add negligible power compared to the 10,000-node multiplier.

### Q16: How does memory partitioning save power?

**A:** A 64KB SRAM as one monolithic block draws full power on every access (all bitlines charge, all sense amps fire). Partitioned into 8 banks of 8KB: only the accessed bank is active (1/8 the bitline + sense amp power). Additional savings: shorter bitlines = lower capacitance per access, smaller decoders, banks in standby can be put in retention or shut down. Trade-off: bank selection logic adds a small amount of area and delay, and the total area increases slightly due to duplicated peripheral circuits. But for memories > 16KB, partitioning almost always wins. Advanced memory compilers offer configurable banking ratios.

### Q17: What are the challenges of near-threshold computing?

**A:** Operating at VDD near Vth (e.g., 0.4V with Vth = 0.35V): (1) Ion/Ioff ratio is very poor (~10-100x) making logic unreliable; (2) Process variation causes huge delay spread (Vth variation of +/-30mV at 3-sigma means some gates are super-threshold and others are sub-threshold simultaneously); (3) SRAM fails first (6T SRAM read/write margins collapse near Vth, need 8T or 10T cells); (4) Frequency drops to MHz range; (5) Leakage becomes comparable to dynamic power, so the energy advantage plateaus. Used in ultra-low-power applications (IoT sensors, biomedical) where throughput is not critical. Design requires wide timing margins, oversized transistors, and error-resilient architectures.

### Q18: Explain the difference between static and dynamic IR drop. Which is worse?

**A:** Static IR drop: caused by average DC current flowing through the resistive power grid. It's the steady-state voltage drop. Calculated as V_drop = I_avg * R_mesh. Typically 10-30mV for a well-designed grid. Dynamic IR drop: caused by sudden current surges (e.g., when the clock edge triggers thousands of FFs switching simultaneously). The transient current causes both resistive drop (I*R) and inductive voltage spikes (L*dI/dt). Dynamic IR drop is typically 2-5x worse than static and can cause 50-100mV droops lasting a few hundred picoseconds. These droops slow down gates exactly when they need to switch, causing timing violations that static analysis misses. This is why dynamic IR drop simulation with realistic switching activity is mandatory for signoff.

### Q19: How does leakage scale across technology nodes?

**A:** Leakage per transistor increased dramatically from 180nm to 65nm due to: thinner gate oxide (more tunneling), shorter channels (worse DIBL), lower Vth (needed to maintain speed as VDD scaled). At 45nm, high-k/metal gate reduced gate leakage significantly. At 22nm, FinFET reduced sub-threshold leakage (better electrostatic control, n closer to 1.0, steeper SS). However, total chip leakage continued to grow because transistor count increased faster than per-transistor leakage decreased. At 7nm, total leakage is ~45-50% of total power for a high-performance design, making leakage management (multi-Vt, power gating) mandatory, not optional.

### Q20: What power verification checks must pass before tapeout?

**A:** (1) UPF consistency: all domains defined, all crossings have level shifters and/or synchronizers; (2) Isolation completeness: every output of every switchable domain is isolated; (3) Retention coverage: all required state registers have retention; (4) Power-aware simulation: correct behavior during all power-on/off transitions; (5) Power grid EM clean: no metal wire exceeds electromigration current density limit; (6) Static IR drop < 5% VDD; (7) Dynamic IR drop < 10% VDD; (8) Rush current within package current limit; (9) Total power within thermal budget at worst-case workload; (10) Leakage power within budget at worst-case temperature; (11) No missing supply connections; (12) All power switches correctly sized and connected.

### Mid-Level Scenario Drills -- Debug Stories You Should Be Able to Tell

Interviews at the 2-5 year level shift from "derive the formula" to "here's a broken
chip/flow -- what do you do?" Practice narrating these five end-to-end.

#### Drill 1: Silicon power is 30% above the signoff estimate

```verilog
Structure the bisection:
1. Idle, clocks gated, nominal V/T -> measures LEAKAGE
   High? -> wrong corner assumption, hotter die than modeled, Vt-mix or
   body-bias misconfiguration, or a domain that never actually power-gates
   (check PMU state residency counters -- a driver holding a vote/wakelock
   on a power domain is the #1 real-world cause)
2. Fixed workload, compare dynamic component
   -> read activity counters: is real utilization what the model assumed?
   -> check CGE in silicon vs RTL estimate (enable telemetry / scan dumps)
   -> check AVS: is the chip running the fused worst-case voltage instead
      of the adaptive one?
3. Still unexplained -> glitch power (signoff used zero-delay activity?)
   or extraction/library model gap -- escalate with a gate-level rerun
   on the measured activity window.
Key behavior: never say "the estimate was wrong" without naming WHICH input
(leakage corner / activity / voltage / glitch) was wrong.
```

#### Drill 2: Chip resets when a big domain wakes up

```verilog
Hypothesis: rush current collapses the shared/parent rail.
Evidence to collect: correlation of resets with wake events; droop monitor
or PMIC undervoltage flags; does slowing the wake (longer switch staging,
trickle-first) make it disappear?
Fixes by layer: more daisy-chain stages / weaker trickle switches; stagger
wakes of multiple domains in firmware (never wake two big domains in the
same 10 us); add decap near the domain; raise parent rail slew headroom.
Signoff lesson learned: ramp-up dynamic IR sim must include the WORST
concurrent-wake scenario, not a lone domain on a quiet die.
```

#### Drill 3: Random single-bit state corruption after sleep/wake cycles

```verilog
Suspects in order:
1. RESTORE pulsed before VVDD stable at the far corner of the domain
   (worse cold: slower ramp) -> voltage detector placement / wake timer
2. Always-on rail droop during the gated period corrupting balloon latches
3. Retention FF library issue: min pulse width on SAVE/RESTORE violated by
   a derated corner
4. Isolation released one cycle early on a bus, letting X-garbage write
   a register file
Discriminator: does corruption pattern follow physical placement (far from
switch chain -> ramp issue) or logical structure (one bus -> sequencing)?
```

#### Drill 4: Leakage passes at signoff, fails 3x over budget in HTOL/burn-in

```verilog
Not a bug -- physics, if signoff corner was optimistic:
leakage doubles every ~10C and burn-in runs hot and high-V.
Check: which corner was the leakage budget defined at? (typical-25C
budgets with FF-125C silicon = 50-100x gap); IDDQ outliers may also be
defects (bridging faults), so separate population shift (process) from
tail outliers (defects). Fix forward: re-fuse AVS/RBB settings for the
hot condition, tighten the Vt mix in the next ECO, or re-negotiate the
budget at a defined corner.
```

#### Drill 5: DVFS transitions occasionally hang the system

```verilog
Classic causes:
1. Frequency raised before voltage settled (sequencing bug or PMIC
   settling-time mis-set for the new board's load) -> setup violations
2. Clock glitch during PLL relock because the divider switch wasn't
   glitch-free -> use a glitchless clock mux, switch to a safe clock
   during relock
3. Level shifter enable race on a domain whose neighbor changed voltage
4. The DVFS governor thrashing between two OPPs at kHz rate (control
   instability) -> add hysteresis/dwell time
Debug data: PMIC telemetry trace + PLL lock signals + last-OPP registers
captured on watchdog reset.
```

---

## Power Analysis and Signoff for Digital IC / ASIC Design

*From [Power_Analysis_and_Signoff.md](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md)*

### Q1: "Walk me through the complete power signoff flow for a tape-out."

**A:** (1) Run gate-level power analysis with post-route extracted SPEF and representative workload SAIF/VCD. Tool: PrimeTime PX or Voltus. Verify average power meets budget per domain. (2) Run static IR drop analysis (Voltus/RedHawk). Verify < 5% Vdd drop at all points. Fix by widening stripes or adding vias where violations occur. (3) Run dynamic IR drop analysis with VCD-based current waveform. Verify < 10% Vdd peak drop. Fix by adding decap cells in hotspot areas. (4) Run EM analysis. Verify all wires meet Jmax for 10-year lifetime at 105C. Widen any violating wires. (5) Feed IR drop map back into STA for voltage-aware timing closure (iterate with ECO if timing fails). (6) Verify leakage at worst-case temperature corner (125C, fast process). (7) Verify thermal stability (no runaway). (8) Document all results and sign off.

### Q2: "Your dynamic IR drop is 15% (above the 10% spec). What do you do?"

**A:** (1) Identify the hotspot region (from IR drop map). (2) Add decoupling capacitor cells in nearby whitespace (filler cells with decap). (3) If insufficient whitespace, resize existing filler cells to decap fillers. (4) Increase power grid density in the hotspot: add more M7/M8/M9 stripes. (5) If the hotspot is at a clock edge (likely), consider clock skew to stagger switching times across the region. (6) Check if the workload is realistic -- if it's a stress test, the actual workload may have lower peak current. (7) As a last resort, add explicit decap macro cells (large MOS caps) but this costs significant area. Re-run dynamic IR drop after each fix to verify improvement.

### Q3: "Explain the difference between SAIF and VCD for power analysis. When do you use each?"

**A:** SAIF captures aggregate statistics (toggle count, time at 0/1) -- compact file, suitable for average power analysis only. Cannot do time-based or peak power analysis. VCD captures every transition with timestamps -- huge files (10-100 GB), but enables time-based power waveforms and peak power identification. Use SAIF for: average power estimation, power budgeting, comparing design alternatives. Use VCD/FSDB for: IR drop analysis (need current waveforms), peak power analysis, power integrity studies, identifying which clock edge causes worst-case current.

### Q4: "A design has 80% SAIF annotation coverage. Is this good enough for signoff?"

**A:** 80% is borderline. The unannotated 20% uses default toggle rates which can be wildly inaccurate. Investigate: if the unannotated 20% is in low-power blocks (configuration registers, rarely-used peripherals), the error is small and 80% may be acceptable. If the unannotated nets are in high-activity datapaths, the error could be large. Target: >90% annotation coverage for signoff. To improve: (1) run longer simulation to exercise more logic, (2) add directed tests for unannotated blocks, (3) use hierarchical annotation (block-level SAIF from unit-level simulation). At minimum, check that the unannotated nets contribute < 5% of estimated total power.

### Q5: "Derive the relationship between IR drop and timing."

**A:** Gate delay T_d ~ C_L * Vdd / (Vdd - Vth)^alpha. With IR drop delta_V, effective Vdd_eff = Vdd - delta_V. New delay: T_d' ~ C_L * Vdd_eff / (Vdd_eff - Vth)^alpha. The fractional delay increase: dT/T = [Vdd_eff/(Vdd_eff-Vth)^alpha] / [Vdd/(Vdd-Vth)^alpha] - 1. For small delta_V: dT/T ~ delta_V * [alpha/(Vdd-Vth) - 1/Vdd]. For Vdd=0.9V, Vth=0.3V, alpha=1.3: dT/T ~ delta_V * [1.3/0.6 - 1/0.9] = delta_V * [2.167 - 1.111] = delta_V * 1.056 per volt = 0.106%/mV. So 50mV IR drop causes ~5.3% delay increase.

### Q6: "What is electromigration and why is it worse at advanced nodes?"

**A:** EM is metal atom displacement by electron wind. Governed by Black's equation: MTTF = A * J^(-n) * exp(Ea/kT). At advanced nodes: (1) wire cross-sections are smaller (thinner metal, narrower width), so current density J increases for the same total current. (2) Copper grain boundaries become a larger fraction of the wire volume (grain boundary EM is worse). (3) Barrier metals consume a larger percentage of the wire cross-section. (4) Some foundries are moving to alternative metals (cobalt, ruthenium) for the thinnest layers, which have different EM characteristics. Mitigation: wider power stripes, more parallel vias, redundant paths, backside power delivery (larger cross-section wires dedicated to power).

### Q7: "Calculate the battery life for a device with 4000mAh battery, 3.7V nominal, and the SoC draws 200mW in light sleep and 1.5W during active use. Usage: 4h active, 20h sleep per day."

**A:** Battery energy = 4000mAh * 3.7V = 14.8 Wh. Daily energy consumption = (1.5W * 4h) + (0.2W * 20h) = 6.0 + 4.0 = 10.0 Wh/day. Battery life = 14.8 / 10.0 = 1.48 days. Note: sleep power (4.0 Wh) is 40% of daily consumption despite being only 200mW -- this is because the device spends 20h in sleep. Reducing sleep power from 200mW to 50mW would save 3 Wh/day, extending battery life to 14.8/7.0 = 2.1 days -- a 42% improvement. This illustrates why standby/sleep power is critical for battery life.

### Q8: "What is thermal runaway? How do you check for it?"

**A:** Thermal runaway occurs when increasing temperature causes more leakage, which causes more heating, in a positive feedback loop. To check: plot P(T) = P_dynamic + P_leakage(T) where P_leakage ~ 2^((T-25)/10), and Q(T) = (T-Ta)/Rth_ja on the same graph. If the curves intersect with dQ/dT > dP/dT at the intersection, the operating point is stable. If P(T) grows faster than Q(T) at all temperatures, there is no stable point and the chip will thermally run away. Modern checks: run power analysis at the signoff temperature, compute junction temperature, re-run power at the new temperature, iterate until convergence. If it diverges, the design has a thermal problem.

### Q9: "How does backside power delivery improve IR drop?"

**A:** Traditional front-side PDN shares the metal stack between power and signals. Top metals (M8/M9/M10) are thick and low-resistance but also carry signals. Backside PDN puts power rails on the wafer backside, accessible through nano-TSVs. Benefits: (1) Power rails can be thicker/wider since they don't compete for routing space -> lower resistance -> lower IR drop (30-50% improvement). (2) Front-side metal can be thinner (less parasitic cap on signals) -> lower dynamic power. (3) More signal routing resource -> less congestion -> shorter wires -> lower wire power. Intel PowerVia (demonstrated at Intel 20A/18A) was the first production implementation.

### Q10: "Walk me through how you would debug a timing failure caused by IR drop."

**A:** (1) Run voltage-aware STA: import IR drop map from Voltus/RedHawk into PrimeTime. (2) Identify failing paths. (3) For each failing path, check which cells see the worst IR drop. (4) If IR drop is localized: add power stripes or decap in that region, re-extract, re-run IR drop and STA. (5) If IR drop is distributed: check overall power grid adequacy -- may need to increase stripe density globally. (6) Check if the power analysis workload is representative -- maybe the VCD shows an unrealistic worst case. (7) If the path barely fails: consider upsizing critical cells (more drive current compensates for lower Vdd) or swapping to LVT. (8) If all else fails: reduce frequency for that operating point or increase voltage (sacrifice power for timing).

### Q11: "What is the difference between peak power and average power? How do you analyze each?"

**A:** Average power is time-averaged over a workload window (mW or W). It determines thermal steady state and battery life. Analyzed using SAIF annotation (aggregate statistics). Peak power is the maximum instantaneous power within a short time window (typically 1-10ns). It determines worst-case IR drop, supply noise, and maximum current from the package. Analyzed using VCD/FSDB (time-resolved switching). Peak can be 3-5x average. Both must be within spec: average for thermal budget, peak for power grid integrity.

### Q12: "How do you budget power for an SoC you haven't designed yet?"

**A:** (1) Start with the total power envelope from the package thermal capability and battery requirements. (2) Allocate top-down based on architecture: CPU ~40%, GPU ~30%, modem ~10%, etc. (base on similar published products or previous generation scaling). (3) For each block, estimate using: published data (e.g., ARM provides power estimates for Cortex cores), historical data from similar blocks scaled by process/voltage/frequency, or high-level models (P = N_gates * alpha * C * V^2 * f). (4) Add margin: 10-20% for unknowns at early stage. (5) Track throughout design: refine at each milestone (RTL, synthesis, layout). (6) If bottom-up numbers exceed top-down budget: either optimize the block or negotiate a larger budget from the system team.

### Q13: "Explain why minimum-power analysis matters for voltage regulator design."

**A:** The voltage regulator must handle the full range from minimum to maximum load current. If max power = 5W at 0.9V (5.56A) and min power = 10mW (11mA), the regulator must: (1) be stable at both extremes (feedback loop must not oscillate), (2) handle the load transient when switching from min to max (requires output capacitance to prevent Vdd droop), (3) maintain voltage accuracy at very low load (some regulators have poor efficiency at light load). The min-to-max ratio (500:1 in this example) determines the regulator architecture (e.g., need a low-power mode for the regulator itself). Knowing min power accurately prevents over-design of the regulator.

### Q14: "A block draws 500mA peak current. The power grid has 10 mOhm resistance from bump to cell. Is this acceptable?"

**A:** Static IR drop = I * R = 500mA * 10mOhm = 5mV. This is only 0.56% of 0.9V -- well within the 5% spec (45mV). However, this is the static case. Dynamic IR drop includes the Ldi/dt component. If the 500mA surges in 100ps with 100pH inductance: V_Ldi/dt = L * dI/dt = 100e-12 * 500e-3/100e-12 = 500 mV. This is catastrophic (56% of Vdd). The solution: decoupling capacitors near the cells to supply the transient current locally, reducing the current that must flow through the inductive path. This is why dynamic IR drop is almost always worse than static, and why decap is essential.

### Q15: "How do you determine the right amount of decap to add?"

**A:** (1) Run dynamic IR drop analysis without extra decap. Identify hotspots and measure the voltage droop depth and duration. (2) For each hotspot, estimate the charge deficit: Q = integral(I_demand - I_supply)dt during the droop event. The decap must supply this charge: C_needed = Q / delta_V_allowed. (3) Check available whitespace near the hotspot. If area allows, insert decap cells totaling C_needed. (4) Re-run dynamic IR drop. If still failing, try: placing decap closer to the demand (proximity matters), or increasing power grid density, or staggering clock edges. (5) Typical total decap: 5-15% of chip area. More is diminishing returns because wire resistance between decap and demand limits effectiveness at high frequency.

### Q16: "What is the impact of process corners on power analysis?"

**A:** Fast process (FF): lower Vth -> higher leakage (worst case for leakage power), higher drive current -> higher dynamic power at same frequency. Slow process (SS): higher Vth -> lower leakage, lower drive current -> need higher Vdd to meet frequency -> might increase dynamic power. Typical (TT): representative for average power reporting. For signoff, you need: FF corner at high temperature (125C) for worst-case leakage, TT corner at nominal conditions for average power budgeting, and SS corner for timing closure (which indirectly affects power through Vdd and frequency choices). Some companies analyze power at 3-5 corners to capture the full range.

---

## Power Fundamentals in Digital IC / ASIC Design

*From [Power_Fundamentals.md](../02_Power_and_Low_Power/01_Power_Fundamentals.md)*

### Q1: "A 28nm design has 10M gates, average switching activity 0.15, average gate capacitance 1.2fF, Vdd=0.9V, 500MHz. Estimate dynamic power."

**A:** P = alpha * C * Vdd^2 * f = 0.15 * (10e6 * 1.2e-15) * 0.81 * 500e6 = 0.15 * 12e-9 * 0.81 * 5e8 = 729 mW. Add ~40% for clock network: ~1.02W total dynamic power. This is the data-path + clock estimate; real chips would also include memory macro dynamic power.

### Q2: "Why does leakage double every ~10C? Derive it."

**A:** I_leak ~ exp(-Vth/(n*kT/q)). As T increases: (1) kT/q increases, directly increasing the exponential argument denominator (but the numerator also decreases because Vth drops with T). For a 10C increase from T to T+10: Vth drops by ~15-20mV, and V_T = kT/q increases by ~0.87mV. The net effect is approximately: I(T+10)/I(T) = exp(delta_Vth/(n*V_T)) ~ exp(15e-3 / (1.3*26e-3)) ~ exp(0.44) ~ 1.6-2.0x. The exact ratio depends on the Vth temperature coefficient and the node. Typical industry rule: 2x per 10C.

### Q3: "Explain why the energy per switching event is C*Vdd^2, not 1/2*C*Vdd^2."

**A:** During 0->1 transition, Vdd delivers C*Vdd^2 energy. Capacitor stores (1/2)*C*Vdd^2. The other half is dissipated in PMOS. During 1->0, the stored (1/2)*C*Vdd^2 is dissipated in NMOS. Per complete cycle: C*Vdd^2 from supply, all dissipated as heat. The (1/2) factor appears only if you count a single transition. The activity factor definition determines which convention you use.

### Q4: "What is DIBL and how does it affect power?"

**A:** DIBL = Drain-Induced Barrier Lowering. In short-channel devices, Vds reduces the source-channel barrier, lowering effective Vth. Higher Vds -> lower Vth -> exponentially more subthreshold leakage. DIBL coefficient eta is ~50-100 mV/V in planar, ~20-30 mV/V in FinFET. This means at Vds=0.8V, effective Vth can be 40-80mV lower than at Vds=0V in planar devices, causing 3-10x more leakage. FinFET's superior electrostatics reduce DIBL significantly.

### Q5: "A chip has 2W leakage at 85C. Estimate leakage at 25C."

**A:** Using the 2x per 10C rule: temperature difference = 60C, so factor = 2^(60/10) = 2^6 = 64. Leakage at 25C = 2W / 64 = 31.25 mW. Note: this is approximate -- the actual ratio depends on process specifics, but 64x is a reasonable estimate.

### Q6: "Why did the industry move to high-k dielectrics?"

**A:** SiO2 gate oxide had to be thinned to ~1.2nm at the 90nm node to maintain gate capacitance (and thus drive current) as channel lengths shrank. At this thickness, direct quantum mechanical tunneling caused gate leakage current of ~100 A/cm^2, making gate leakage comparable to subthreshold leakage. HfO2 (k~22 vs 3.9 for SiO2) allows a physically thicker oxide (e.g., 4nm HfO2 ~ 0.7nm SiO2 equivalent) with the same capacitance, reducing tunneling current by ~100x. Intel introduced high-k/metal gate at 45nm (2007).

### Q7: "What is the stack effect and how does it help leakage?"

**A:** In series-connected transistors (e.g., NAND pull-down), when one transistor is off, the intermediate node floats to a voltage between 0 and Vdd. This causes the off transistor to have: (1) positive Vsb -> higher Vth via body effect, and (2) reduced Vds -> reduced DIBL. Both effects reduce leakage. A 2-stack reduces leakage by ~10x compared to a single transistor. 3-stack gives ~25-50x reduction. This is why NAND gates are preferred over NOR gates for low-leakage design (NMOS stack in NAND benefits from stack effect more than PMOS stack in NOR because PMOS has higher Vth already).

### Q8: "What fraction of switching power is due to glitches in a typical design?"

**A:** 15-30% in unoptimized datapaths (e.g., ripple-carry adders, multiplier trees). After optimization (path balancing, retiming), this can be reduced to 5-10%. Clock gating downstream logic eliminates the power impact of glitches on registered outputs but not on the combinational logic itself. Synthesis tools automatically minimize glitches through technology mapping, but physical design (routing delays) can introduce new glitch paths.

### Q9: "Compare leakage mechanisms: subthreshold vs gate vs junction vs GIDL. Which dominates when?"

**A:** At room temperature, 7nm: subthreshold >> gate (mitigated by high-k) > GIDL > junction. At 125C: subthreshold >>> junction (grows fastest with T) > GIDL > gate (temperature-insensitive). For HVT cells at room temp: subthreshold is suppressed, GIDL can become comparable. For ultra-low-voltage near-threshold designs: subthreshold dominates overwhelmingly (exponentially sensitive to Vgs near Vth).

### Q10: "Explain forward vs backward SAIF annotation."

**A:** Forward annotation: simulate RTL, dump SAIF at RTL level, then map net names from RTL to gates during power analysis. This requires name mapping (can be lossy). Backward annotation: tools read the gate-level netlist, generate a "SAIF template" with gate-level net names, you simulate the gate-level netlist (or annotated RTL) and fill in the template. Backward is more accurate because net names match exactly. Forward is faster because RTL simulation is much faster than gate-level simulation.

### Q11: "How would you estimate the power of a block you haven't designed yet?"

**A:** (1) Use a similar block from a previous project as a reference and scale by gate count, activity, Vdd, and frequency. (2) Use high-level estimation: P = alpha * N_gates * C_avg * Vdd^2 * f, where N_gates and C_avg come from synthesis estimates or published data. (3) For memory-dominated blocks, estimate memory macro power from datasheets and add logic power. (4) Industry rules of thumb: ARM Cortex-A class cores are ~100-300 mW/GHz at 7nm; DSP blocks are ~50-100 mW/GHz; always-on domains target < 1 mW.

### Q12: "What is the difference between average power and peak power? Why do both matter?"

**A:** Average power (mW over a workload period) determines thermal steady state and battery life. Peak power (instantaneous maximum, often over a 1-10ns window) determines: (1) worst-case IR drop on the power grid (V_drop = I_peak * R_grid), (2) Ldi/dt noise on the supply, (3) maximum current the package must deliver. Peak can be 3-5x average. Both must meet specs: average for thermal/battery, peak for supply integrity.

### Q13: "If you reduce Vdd by 10%, what happens to dynamic power, leakage, and frequency?"

**A:** Dynamic power: P ~ Vdd^2, so 0.9^2 = 0.81 -> 19% reduction. Leakage: subthreshold leakage has weak Vdd dependence (mainly through DIBL: lower Vds -> slightly higher Vth -> less leakage, maybe 10-20% reduction, highly process-dependent). Gate leakage reduces roughly proportionally with Vdd. Frequency: Fmax ~ (Vdd-Vth)^alpha / Vdd where alpha~1.3. For Vdd=0.9V->0.81V, Vth=0.3V: (0.81-0.3)^1.3/0.81 vs (0.9-0.3)^1.3/0.9 -> roughly 15-20% frequency reduction. Net: 19% power savings for ~15-20% performance loss -- this is why DVFS is so effective.

### Q14: "Why is clock power so high?"

**A:** Clock toggles every cycle (activity factor = 1.0, vs 0.05-0.2 for data). Clock networks are heavily buffered (thousands of buffers in clock tree) with large drive strengths. Clock wires run long distances across the chip with significant wire capacitance. In a typical SoC: clock tree = 30-50% of dynamic power. This is why clock gating is the single most impactful power reduction technique.

### Q15: "What is the minimum energy point (MEP) and why does it exist?"

**A:** As you reduce Vdd, dynamic power drops quadratically, but the circuit slows down, so you need more time (cycles) to do the same work. At very low Vdd (near Vth), the circuit is so slow that leakage energy per operation dominates (leakage power * long execution time). The MEP is the Vdd where total energy per operation (dynamic + leakage) is minimized. It typically occurs at Vdd ~ Vth (near-threshold computing). Below MEP, reducing Vdd actually increases total energy because the performance loss causes leakage to dominate. MEP is ~0.3-0.4V for most processes.

### Q16: "Explain the power-performance-area (PPA) trade-off at the cell level."

**A:** A larger cell (wider transistors) has: more drive current -> faster (better P), but more capacitance -> more switching power (worse P), and more area (worse A). Multi-Vt: LVT is faster (better performance) but leaks ~5-20x more than HVT (worse leakage power), at the same area. Sizing: upsizing a cell on the critical path improves timing but increases its capacitance and the capacitance it presents to its driver. The optimization is a constrained problem: minimize power subject to timing constraints, using cell sizing and Vt assignment as the primary levers.

---

## Power Reduction Techniques for Digital IC / ASIC Design

*From [Power_Reduction_Techniques.md](../02_Power_and_Low_Power/03_Power_Reduction_Techniques.md)*

### Q1: "A block has 5000 FFs, clock freq 1GHz, Vdd=0.8V, C_clk per FF = 8fF. Calculate power savings with 75% clock gating efficiency."

**A:** P_saved = CGE * N * C_clk * Vdd^2 * f = 0.75 * 5000 * 8e-15 * 0.64 * 1e9 = 0.75 * 5000 * 8 * 0.64 * 1e-6 = 0.75 * 25,600 * 1e-6 = 0.75 * 25.6 mW = 19.2 mW. With ICG overhead (assume 100 ICG cells at 10uW each = 1mW), net savings = ~18.2 mW.

### Q2: "Why must you reduce frequency before voltage when scaling down?"

**A:** If you reduce voltage first while still running at high frequency, the gate delay increases (Td ~ Vdd/(Vdd-Vth)^alpha). If the new delay exceeds the clock period, setup time violations occur, causing functional failures (wrong data captured by flip-flops). Reducing frequency first ensures that even at the slower voltage, all timing constraints are met. The reverse applies when scaling up: increase voltage first to provide timing headroom for the higher frequency.

### Q3: "Explain the difference between isolation types and when to use each."

**A:** AND isolation (clamp to 0): use for data buses, address buses -- zero is a benign default that won't trigger downstream logic. OR isolation (clamp to 1): use for active-low signals like reset_n, enable_n -- clamping to 1 means "deasserted" which is the safe state. Latch isolation (hold last value): use for status/configuration outputs where the downstream logic should see the last valid value, not a forced constant. Clamp-high/low: simpler versions that tie to rail -- lower area than latch isolation but less flexible.

### Q4: "Design a power-up sequence for a CPU core with retention. What happens if you get the order wrong?"

**A:** Correct sequence: (1) turn on power switch (staged), (2) wait for voltage stable, (3) assert reset, (4) restore retention, (5) deassert reset, (6) deassert isolation, (7) enable clocks. If you deassert isolation before restoring retention: the flip-flops have random state, which propagates to always-on domain causing potential functional errors or bus contention. If you restore before voltage is stable: retention restore requires correct voltage to properly transfer data from shadow to master latch -- may get corrupted bits. If you enable clocks before deasserting reset: flip-flops clock in garbage data for one or more cycles.

### Q5: "A design has 80% HVT, 15% SVT, 4% LVT, 1% ULVT. Total 5M gates. Leakage per gate: HVT=1nA, SVT=5nA, LVT=25nA, ULVT=100nA. Vdd=0.75V. Calculate total leakage power."

**A:**
```verilog
HVT:  4.0M * 1nA   = 4.0 mA
SVT:  0.75M * 5nA  = 3.75 mA
LVT:  0.2M * 25nA  = 5.0 mA
ULVT: 0.05M * 100nA = 5.0 mA
Total: 17.75 mA * 0.75V = 13.3 mW at 25C

At 85C (6x factor from 25C, using 2x per 10C):
  13.3 * 64 = 851 mW ... using 2^((85-25)/10) = 2^6 = 64x
  Actually that's the right math. 851 mW at 85C.

Note: 1% ULVT cells contribute 28% of total leakage current!
```

### Q6: "What is the stack effect? Quantify its leakage reduction."

**A:** When NMOS transistors are stacked in series (like a NAND gate), and one transistor is OFF, the intermediate node floats to a voltage Vm (typically 50-200mV). This causes: (1) the OFF transistor has positive Vsb, increasing its Vth by gamma*delta (maybe 30-50mV), (2) the OFF transistor has reduced Vds (only Vm instead of Vdd), reducing DIBL. Combined: 2-stack reduces leakage by ~5-10x vs single transistor. 3-stack: ~15-30x. This is why NAND gates leak less than equivalently-sized inverters, and why 4-input NAND gates are sometimes used in non-critical paths for leakage reduction.

### Q7: "Compare power gating header vs footer switches."

**A:** Header (PMOS between Vdd and logic): cleaner ground for internal logic (no ground bounce), better noise margins for retention registers, larger area (PMOS is ~2x wider for same current). Footer (NMOS between logic and GND): smaller area (NMOS has higher mobility), but virtual ground bounces during switching causing noise on internal nodes. Industry standard: headers for most ASIC designs. Footers sometimes for SRAM arrays (regular structure, easier to manage ground bounce). For FinFET: the PMOS/NMOS mobility gap is smaller (both use undoped channel), so the area advantage of NMOS footer is reduced.

### Q8: "How does AVS save power compared to fixed DVFS?"

**A:** DVFS uses a fixed voltage for each frequency point, designed for the worst-case silicon (slowest chip). Fast silicon runs at unnecessary high voltage. AVS measures actual silicon speed (via ring oscillator or critical path monitor) and reduces voltage until just meeting timing. Typical savings: 50-100mV on fast silicon. For Vdd=0.9V, saving 75mV reduces dynamic power by 1-(0.825/0.9)^2 = 1-0.840 = 16%. This adds up across billions of chips. Apple, Qualcomm, and all major mobile SoC vendors use AVS.

### Q9: "What determines the minimum power gating block size? When is power gating not worth it?"

**A:** Overhead: switch cells (~3-5% area), isolation cells (per output signal, ~1 gate each), retention FFs if needed (~2x area each), control logic, design/verification effort. Break-even: if the block's leakage savings during sleep exceeds the overhead. For a tiny block (100 gates), the switch/isolation/control overhead exceeds the leakage savings -- not worth it. Rule of thumb: power gating is beneficial for blocks > ~10K gates that are idle > 50% of the time. The minimum idle time for break-even depends on the power-up energy cost (charging internal capacitance).

### Q10: "Explain operand isolation. Give an example where it saves significant power."

**A:** Operand isolation freezes the inputs of a combinational block when its output is not needed, preventing useless toggling. Classic example: a CPU with an ALU and a multiplier behind a result MUX. When the instruction is ADD, the multiplier inputs should be frozen (AND-gated with the MUL select signal) to prevent the multiplier from burning dynamic power computing an unused result. For a 64-bit multiplier that consumes 15mW and is used only 20% of cycles: savings = 0.80 * 15 = 12 mW. Synthesis tools can do this automatically with pragmas or automatic operand isolation inference.

### Q11: "What is the energy overhead of power-gating on/off transitions?"

**A:** Each power-on event requires charging the internal decoupling capacitance from 0 to Vdd: E_charge = C_int * Vdd^2 (just like switching energy). For C_int = 10nF, Vdd = 0.8V: E = 10e-9 * 0.64 = 6.4 nJ. If the domain sleeps for T_sleep and saves P_leak during sleep: break-even when P_leak * T_sleep > E_charge. For P_leak = 10 mW: T_sleep_min = 6.4nJ / 10mW = 640 ns. So if the block sleeps for less than ~640ns, it's not worth power-gating (you spend more energy on the transition than you save). This sets the minimum granularity of power gating decisions.

### Q12: "Compare DVFS, clock gating, and power gating in terms of power savings, granularity, and latency."

**A:**

| Aspect        | Clock Gating       | DVFS               | Power Gating         |
|---------------|--------------------|--------------------|----------------------|
| Saves         | Dynamic (clock)    | Dynamic (Vdd^2*f)  | Leakage + dynamic    |
| Granularity   | Per-register group | Per-voltage-domain  | Per-power-domain     |
| Latency       | 0 cycles           | 10-100 us          | 5-50 us + restore    |
| Overhead      | ~5% area (ICGs)    | PMIC, PLL, LDO     | Switches, ISO, ret   |
| When to use   | Always (free win)  | Workload varies     | Block fully idle     |

### Q13: "What is near-threshold computing and why is it energy-efficient?"

**A:** Near-threshold computing operates at Vdd ~ Vth (0.3-0.5V). Dynamic energy drops as Vdd^2 (huge savings: (0.4/0.9)^2 = 0.20 -> 80% less). But speed drops dramatically (circuit is ~10x slower). Energy per operation: E = P * T = C*Vdd^2 (dynamic, same per op) + P_leak * T_op. At very low Vdd, T_op is huge and leakage energy dominates. The minimum energy point (MEP) is typically at Vdd ~ Vth. Below MEP, total energy actually increases. Near-threshold is ideal for energy-constrained, latency-tolerant applications: IoT sensors, biomedical implants, always-on wake-up circuits.

### Q14: "How do you handle clock gating for multi-bit registers that span different enable conditions?"

**A:** If bits [31:16] have enable A and bits [15:0] have enable B, the synthesis tool inserts two separate ICG cells. If bits [31:24] share enable A and bits [23:0] have enable B, you get two ICGs with different fan-out. The tool optimizes by grouping FFs with the same enable into one ICG (up to max_fanout). Sometimes restructuring RTL to align enable boundaries with byte/word boundaries improves CGE. The -minimum_bitwidth constraint prevents insertion of ICG for very small groups (e.g., a single FF) where the ICG power overhead exceeds the savings.

### Q15: "A 7nm chip runs at 2GHz, 0.75V. You need to reduce power by 30% without changing the design. What do you do?"

**A:** Options: (1) Reduce voltage to ~0.64V (0.64/0.75)^2 = 0.73 -> 27% dynamic power reduction, but frequency drops to maybe 1.4-1.5 GHz. If throughput matters, might need to accept the slowdown or compensate with parallelism. (2) Reduce frequency to 1.4 GHz (0.7x) -> 30% dynamic savings at same voltage, but no leakage savings. (3) DVFS: reduce to 0.68V/1.5GHz -> ~30% dynamic savings with proportional frequency hit. (4) If design allows: aggressive clock gating analysis to find ungated FFs (tools like CG audit can find 5-15% more gating opportunities). (5) Multi-Vt re-optimization: if current design has too many LVT cells, re-optimize with tighter constraints to trade some timing margin for less leakage. Best approach: combination of (3) DVFS + (4) improved clock gating + (5) Vt re-optimization.

### Q16: "Explain retention register timing requirements in detail."

**A:** The SAVE signal must be pulsed AFTER the last clock edge (so all FFs have valid data) and BEFORE power is removed. The SAVE pulse width must meet the shadow latch setup and hold time (typically 200-500ps minimum pulse width). The RESTORE signal must be pulsed AFTER power is stable (voltage within 90-95% of nominal) and BEFORE the first functional clock edge. RESTORE also has minimum pulse width requirements. Between SAVE and RESTORE, the shadow latch must retain data on the always-on supply -- any voltage droop on the always-on rail can corrupt retained state. The always-on supply must remain stable within ~10% of nominal throughout the power-gated period.

### Q17: "Emulation shows 35% CGE where architecture predicted 80%. Walk me through your debug."

**A:** First separate structural ICG coverage (% FFs behind an ICG -- if this is low, synthesis settings or RTL coding style are the problem) from CGE (% cycles gated -- if coverage is high but CGE is low, the ENABLES are weak). Then rank ICGs by wasted clock power (fanout x %enabled) and inspect the top offenders: valids that free-run regardless of downstream readiness, recirculation muxes the tool failed to extract (often due to reset-priority coding), missing module-level gating so the upper clock tree toggles even when leaf ICGs gate, and over-fine ICGs whose own overhead exceeds savings. Also confirm the workload: CGE measured on a busy benchmark says nothing about idle power. Fixes: qualify enables at the source, recode enable patterns, add hierarchical ICGs from FSM-idle signals. Verify the fix shows up as reduced CLOCK TREE power, not just register power.

### Q18: "Explain the clock-gating check in STA. Why does a plain AND gate fail it?"

**A:** The gating signal may only change while the clock is low: setup is checked against the next rising edge, hold against the falling edge. With a plain AND gate and an enable launched from a posedge FF, the enable changes just after the rising edge -- while the clock is high -- truncating or glitching the pulse: a hold violation by construction. The ICG's transparent-low latch fixes this: it is opaque during the high phase (blocks mid-pulse changes) and closes at the rising edge, giving the enable nearly a full cycle to arrive. Many teams additionally require enable arrival by the falling edge as a margin policy; that is a house rule, not the hard requirement.

### Q19: "Compare a board PMIC, an integrated buck (FIVR), and a digital LDO (DLVR) for per-core DVFS."

**A:** Board PMIC: efficient (85-92%) but slow (10-100 us) and one rail feeds many cores -- voltage is set by the most demanding core. FIVR: on-die buck with package inductors, ~us transitions, per-domain rails, but inductor cost/area and conversion loss. DLVR (Intel Core Ultra: per P-core, per E-core cluster, per ring): a digital linear regulator -- nanosecond-class response and very fine granularity, but linear efficiency = Vout/Vin, so it only wins when shaving a small delta off the shared rail; a bypass mode shorts it when the core needs full voltage. The architectural point: per-core regulation converts the (V_shared - V_needed) gap on every idle/slow core into real power savings, which a single rail can never capture.

### Q20: "A voltage droop event happens in 5 ns. Your firmware DVFS loop runs every millisecond. Reconcile."

**A:** Power management is a reaction-time hierarchy: ns-scale first droop is handled by circuits -- on-die droop detectors plus adaptive clock stretching (AMD: DLL-based phase insertion; IBM POWER9 adaptive clocking) that lengthen the clock period through the droop, plus local decap; us-scale is the regulator's control loop and load-line; ms-scale is firmware DVFS adjusting the operating point; seconds-scale is thermal management. Each layer exists because the faster phenomenon cannot wait for the slower controller. The payoff of the ns layer is guardband recovery: without it you carry a permanent ~50 mV margin (~11% dynamic power at 0.9V) for an event that occurs rarely and lasts tens of ns.

### Q21: "An 8MB L2 is idle 60% of the time. Which SRAM mode do you use and why not the deeper one?"

**A:** Light sleep for short idle windows (1-2 cycle wake keeps it transparent to the pipeline), deep-sleep retention for sustained idle (data retained at ~0.5-0.6V array voltage, ~70-90% leakage saved, but wake is 100s of ns -- needs a predictor or explicit driver hint to hide). Shutdown only if the contents are clean or worth flushing: wake costs a flush-and-refill (us-to-ms of effective penalty and DRAM traffic energy). The decision is a break-even calculation between leakage saved per idle window and the energy/latency of entry/exit -- same math as power-gating break-even, plus the retention-Vmin margin question at temperature extremes.

### Q22: "Describe a leakage-recovery ECO and three ways it can go wrong."

**A:** Post-route, swap positive-slack cells to higher-Vt footprint-compatible variants (PrimeTime fix_eco_power -> ICC2/Innovus ECO; or Innovus optPower -postRoute), iterating with incremental STA; typically 20-50% leakage reduction for free. Failure modes: (1) swapping to exactly-zero slack leaves nothing for OCV updates and aging -- BTI raises Vt over life, so end-of-life paths fail; (2) corner blindness -- a swap that closes at the nominal corner violates at the low-voltage corner where HVT delay blows up, or busts the high-temp leakage budget; (3) touching the wrong cells -- clock network cells (skew shift) or clock-gate enable paths (tight checks). Also watch max_transition: HVT variants have weaker drive, and a legal-timing swap can still create a slew violation downstream.

### Q23: "How are multiple Vt flavors built on a GAA nanosheet process if the channel is undoped and there's no room for thick work-function metal?"

**A:** Dipole engineering: ultra-thin dipole layers (separate n-type and p-type dipole materials) inserted in the high-k gate stack shift the effective work function without consuming the few nanometers between stacked sheets. TSMC's N2 reports six Vt levels across ~200 mV using third-generation dipole integration. Nanosheets also add sheet-WIDTH tuning (NanoFlex short vs tall cells) as a quasi-continuous drive knob alongside discrete Vt. Interview-relevant consequence: early GAA PDKs offer fewer Vt/Lg combos than mature FinFET nodes, so leakage recovery has less room and Vt-flavor mistracking across corners must be signed off explicitly.

### Q24: "Why do AI training clusters create a power-delivery problem that single-chip DVFS can't solve, and what's done about it?"

**A:** Training synchronizes thousands of GPUs: all compute bursts and all communication stalls happen in lock-step, so the facility load swings by tens of percent of MW-scale power at millisecond granularity -- a grid/facility problem, not a chip problem. Chip-level DVFS reacts per-GPU and actually AMPLIFIES synchronization. Mitigations operate at the rack/system level (NVIDIA GB300 NVL72 as the canonical example): energy storage in the power shelves (~65 J of capacitance per GPU) absorbs spikes, ramp-rate-limited power caps make job starts gradual, and a controlled "burn" mode dissipates power on abrupt job end so the load ramps down smoothly -- together cutting peak grid demand by up to ~30%. Knowing this bridges chip power techniques to data-center reality, which is exactly what AI-hardware roles screen for.

---

## UPF (Unified Power Format) Power Intent Specification

*From [UPF_Power_Intent.md](../02_Power_and_Low_Power/04_UPF_Power_Intent.md)*

### Q1: "What is UPF and why can't you capture power intent in RTL?"

**A:** UPF (IEEE 1801) is a TCL-based format that specifies power domains, supply networks, power states, and low-power cell insertion strategies. RTL describes WHAT the circuit does functionally, not HOW it is powered. You cannot express "this block can be powered off" or "these registers retain state" in Verilog/VHDL. UPF separates power intent from function, allowing the same RTL to be implemented with different power strategies and enabling all EDA tools to consume a single, consistent power specification.

### Q2: "Walk me through writing UPF for a design with two power-gatable blocks."

**A:** (1) create_power_domain for each block and the top-level always-on domain. (2) create_supply_net for VDD (always-on) and switched supply nets. (3) create_power_switch for each gated domain mapping VDD to VDD_switched. (4) set_isolation on outputs of each gated domain with appropriate clamp values. (5) set_isolation_control with the control signal and sense. (6) If retention needed: set_retention and set_retention_control. (7) set_level_shifter if voltage domains differ. (8) add_power_state for each supply set. (9) create_pst and add_pst_state for legal state combinations. See the complete example in Section 3.

### Q3: "What is the difference between isolation and level shifting?"

**A:** Isolation handles the ON/OFF power state transition -- it clamps outputs to a known value when the source domain is powered off. Level shifting handles VOLTAGE DIFFERENCES between domains that are both on but at different voltages. You might need BOTH: a combined isolation-level-shifter cell that can clamp when the source is off AND level-shift when it is on at a different voltage. Some cell libraries have combo cells (ISO+LS) to save area.

### Q4: "How does power-aware simulation differ from regular RTL simulation?"

**A:** In regular simulation, all signals have defined values regardless of power state. In power-aware simulation, the simulator reads UPF, tracks supply state, and forces all logic in a powered-off domain to X. Isolation cells replace X with their clamp value. Retention registers hold saved values even when their domain is X. This catches bugs like: missing isolation (X propagates to always-on domain), wrong clamp value (downstream logic gets wrong constant), incorrect power sequencing (restore before power is stable). These bugs are INVISIBLE to regular simulation.

### Q5: "Explain supply set handles and successive refinement."

**A:** Supply set handles are abstract placeholders for {power, ground} connections. An IP block's UPF uses handles (PD.primary.power, PD.primary.ground) without naming actual physical nets. At SoC integration, the integrator connects these handles to real supply nets. This allows the IP to be reused in different SoCs with different supply topologies. Successive refinement: write high-level UPF at RTL (domains, strategies), refine at synthesis (cell types, constraints), refine again at physical design (switch placement, grid design). Each stage adds detail without rewriting previous stages.

### Q6: "What are the most common UPF bugs you've seen?"

**A:** (1) Missing isolation on qualified outputs (signal that goes through AND gate before leaving domain -- still needs isolation). (2) Wrong clamp value: active-low control clamped to 0 (asserted) instead of 1 (deasserted). (3) Missing level shifter on cross-voltage paths. (4) Incorrect power sequence: isolation after power-off instead of before. (5) No always-on buffers for clock/control signals inside gated domain. (6) Retention restore before voltage is stable. (7) Forgetting that combinational paths through a gated domain also need isolation (not just registered outputs).

### Q7: "What is a power state table (PST) and why is it important?"

**A:** The PST defines all LEGAL combinations of power states across domains. Example: it is legal for CPU to be ON and PERI to be OFF, but it is ILLEGAL for TOP to be OFF while CPU is ON (because CPU depends on TOP for bus fabric and interrupts). Tools use the PST to: (1) verify that the power management FSM only generates legal state transitions, (2) analyze power at each state, (3) verify that isolation/retention is properly handled for each transition. Missing a legal state in the PST means tools won't check it; including an illegal state means tools might allow an invalid configuration.

### Q8: "How do you handle a signal that goes from domain A through domain B to domain C, where B can be powered off?"

**A:** This is a "feed-through" signal. When domain B is off, the signal path through B is broken (buffers in B are dead). Options: (1) Route the signal AROUND domain B (bypass at the top level). (2) Use always-on buffers in domain B for the feed-through path (mark in UPF as always-on elements). (3) If the signal is not needed when B is off, isolate at both the A->B and B->C boundaries. Option 2 is most common: the feed-through signal uses always-on buffers so it works regardless of B's power state.

### Q9: "Explain the difference between UPF 1.0 and UPF 2.0."

**A:** UPF 1.0 uses direct supply net connections (connect_supply_net to domain). UPF 2.0 introduces supply SETS (abstract bundles of power+ground) with HANDLES. This enables: (1) successive refinement (abstract at IP, concrete at SoC), (2) cleaner expression of multi-rail designs, (3) supply expressions for power states (voltage-based state definitions), (4) better IP delivery model. UPF 2.0 also improved the power state table mechanism and added explicit support for retention supply specification.

### Q10: "What Liberty attributes must a retention register have?"

**A:** `retention_cell` attribute identifying the save/restore mechanism. `pg_pin` with `pg_type : backup_power` for the always-on supply pin (VDDAO). `retention_pin` attributes on the SAVE and RESTORE pins indicating their function and active level. The cell must have timing arcs for save and restore operations (setup/hold of data to save edge, minimum pulse width of save/restore). The `power_down_function` attribute must correctly describe behavior when the primary supply is off.

### Q11: "How do you verify that ALL cross-domain signals have proper isolation or level shifting?"

**A:** (1) Static UPF checking tools (VC LP, Conformal LP) automatically trace all signals crossing domain boundaries and flag any that lack isolation or level shifting. (2) Power-aware simulation: signals without isolation show X propagation. (3) Manual review: generate a cross-domain signal list from synthesis and review each one. (4) Formal verification: some tools can formally prove that no X can reach an always-on domain under any power state sequence. Best practice: run static checks early (at UPF creation time) and continuously during design.

### Q12: "What is the role of the power management unit (PMU) and how does it relate to UPF?"

**A:** The PMU is the hardware block that generates the control signals referenced in UPF: sleep_n (power switch control), iso_en (isolation control), save/restore (retention control). The PMU implements the power state machine (transitions defined by the PST). UPF specifies WHAT should happen (which domain, which cells); the PMU implements WHEN and HOW the transitions occur. The PMU itself must be in the always-on domain. Its design must match the sequencing requirements implied by the UPF (e.g., isolation before power-off, restore after power-up).

### Q13: "Can two domains with different power states share combinational logic?"

**A:** No. Every gate belongs to exactly one power domain and is powered by that domain's supply. If a combinational cone spans two domains, it must be split: each portion is in its own domain, with isolation at the boundary. In practice, synthesis handles this by replicating logic or inserting isolation at the appropriate point. If you WANT shared logic between domains that can independently power down, put the shared logic in the always-on domain with its own power supply.

### Q14: "How do you handle UPF for an IP block that you receive as a hard macro?"

**A:** The IP vendor should deliver: (1) an abstract UPF model describing the IP's power domains and supply requirements, (2) Liberty (.lib) with power pin definitions (pg_pin attributes), (3) LEF with power rail locations. The SoC integrator writes top-level UPF that: creates supply nets and connects them to the IP's supply ports, defines isolation for signals crossing between the IP and the rest of the SoC, and includes the IP's power states in the system PST. If the vendor doesn't provide UPF, you must reverse-engineer the power structure from the .lib and documentation.

### Q15: "Explain how UPF handles retention for a design where only 10% of flip-flops need retention."

**A:** Use selective retention with the -elements flag:

```tcl
set_retention CPU_RET \
    -domain PD_CPU \
    -elements {cpu_core/reg_file cpu_core/pc_reg cpu_core/status_reg} \
    -retention_power_net VDD \
    -retention_ground_net VSS
```

Only the specified registers are replaced with retention variants. The other 90% remain standard flip-flops and lose state when powered off. After power-up, the retained registers have their saved values; the non-retained registers must be re-initialized (via reset or software). This saves ~90% of the retention area and always-on leakage overhead compared to retaining all FFs. The trade-off is longer wake-up time (need to re-initialize the non-retained state).

