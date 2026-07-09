# 06 — Signoff: Interview Questions

Consolidated interview Q&A and worked problems from every page in `06_Signoff/`, moved here to keep topic pages focused on concepts. Each section links back to its source page for full context.

---

## Design for Testability (DFT) and ATPG -- Senior Engineer Deep Dive

*From [DFT_and_ATPG.md](../06_Signoff/02_DFT_and_ATPG.md)*

### Q1: What is the purpose of a lockup latch in a scan chain?

**A:** A lockup latch is inserted between scan flip-flops that are in different
clock domains (or different edges of the same clock) within a scan chain. During
shift mode, data must flow reliably from one FF to the next on every clock edge.
If two adjacent FFs in the scan chain are clocked on different edges (e.g.,
posedge and negedge), the data launched by the first FF may not meet the
setup/hold requirements of the second FF because the time between the active
edges is too short (nearly zero in worst case). The lockup latch (a transparent
latch clocked by the source domain's clock) adds a half-cycle buffer, ensuring
the downstream FF always has adequate setup time. The latch is transparent
during the active phase of the source clock and holds the value during the
inactive phase, effectively time-shifting the data.

### Q2: Why can't stuck-at testing alone guarantee a working chip at speed?

**A:** Stuck-at faults model logical correctness but not timing. A gate with a
resistive defect (e.g., partial via open, weak transistor) may produce the
correct logic value but too slowly. The chip passes stuck-at tests (which are
applied at slow tester speed) but fails at the operational frequency because
the signal doesn't arrive before the clock edge. Transition delay fault (TDF)
testing addresses this by using two-pattern tests applied at functional speed,
detecting gates that are "slow-to-rise" or "slow-to-fall."

### Q3: Explain the difference between LOS and LOC at-speed testing.

**A:** In **Launch-Off-Shift (LOS)**, the last shift clock serves as the launch
clock. The test requires scan_enable to transition from HIGH to LOW between the
launch and capture clocks -- within one functional clock period. This gives
ATPG more control over the V2 pattern (it's V1 shifted by one position) and
typically yields higher TDF coverage. However, the tight SE timing requirement
is challenging for high-frequency designs.

In **Launch-Off-Capture (LOC/Broadside)**, after shifting completes and SE goes
LOW (with ample time margin), two at-speed functional clock pulses are applied.
The first pulse launches the transition (its response becomes V2), and the
second captures the result. SE timing is relaxed, but V2 is determined by the
circuit's functional response to V1, limiting ATPG's controllability. LOC is
more commonly used in industry due to its simpler SE timing requirement.

### Q4: What is fault collapsing and why is it important?

**A:** Fault collapsing reduces the fault list by removing equivalent and
dominated faults. Two faults are equivalent if no test can distinguish them
(they produce identical faulty behavior for all inputs). A fault f1 dominates
f2 if every test for f2 also detects f1. By the checkpoint theorem, we only
need to test faults at primary inputs and fanout branches to cover all faults.
Collapsing typically reduces the fault list by 50-70%, dramatically reducing
ATPG runtime and pattern count.

### Q5: How does test compression work and what is a typical compression ratio?

**A:** Test compression uses an on-chip decompressor (typically LFSR + phase
shifters) to expand a few external tester channels into hundreds of internal
scan chains, and a compactor (XOR network) to compress hundreds of chain
outputs back to a few tester channels. This works because for any given test
pattern, only 1-5% of scan cells need specific values; the rest are don't-care.
The LFSR-based decompressor fills don't-cares with pseudo-random values while
precisely controlling the specified bits. Typical compression ratios are
100x-200x, reducing test data from gigabytes to megabytes.

### Q6: What are random pattern resistant faults and how are they addressed in LBIST?

**A:** Random pattern resistant (RPR) faults require specific input combinations
that have very low probability in pseudo-random sequences. For example, an
N-input AND gate's output SA0 fault requires all N inputs = 1, with probability
2^(-N). For N=20, you'd need ~1M random patterns just for this one fault.

Solutions: (1) Test point insertion -- adding control points (AND/OR gates with
scan FFs) that bias hard-to-control signals, and observation points (extra scan
FFs) on hard-to-observe signals. (2) Weighted random patterns -- biasing LFSR
outputs toward required values. (3) Deterministic BIST -- storing a small set
of deterministic seed values that the LFSR expands into targeted patterns for
RPR faults.

### Q7: Explain March C- algorithm and its fault coverage.

**A:** March C- consists of 6 elements with 10N total operations. It detects
stuck-at faults, transition faults, address decoder faults, and most coupling
faults (CFin, CFid) but not state coupling faults (CFst) or neighborhood
pattern sensitive faults (NPSF). The ascending and descending address orders
in elements 1-4 ensure that coupling between any two cells is tested in both
directions. The 10N complexity makes it practical for large memories while
providing good fault coverage. For more comprehensive testing, March SS (22N)
adds CFst and NPSF detection.

### Q8: What is the aliasing problem in MISR-based signature analysis?

**A:** Aliasing occurs when a faulty circuit accidentally produces the same MISR
signature as the fault-free circuit. Since the MISR compresses a large amount of
response data into an n-bit signature, information is lost. The probability of
aliasing is 2^(-n). For a 32-bit MISR, this is approximately 2.3 x 10^(-10),
or about 0.23 parts per billion -- negligibly small for practical purposes.

### Q9: How does cell-aware testing differ from traditional fault models?

**A:** Traditional stuck-at and TDF models only consider faults at cell
input/output pins. Cell-aware testing models defects INSIDE standard cells
(transistor-level opens, shorts, resistive connections) using SPICE-based
characterization of each library cell's physical layout. At nodes below 28nm,
intra-cell defects become a significant contributor to test escapes. Cell-aware
tests typically add 5-15% more patterns beyond TDF and catch 2-5% additional
silicon defects.

### Q10: Describe the JTAG TAP controller and its key states.

**A:** The TAP controller is a 16-state FSM driven by TMS and TCK. Key states:
Test-Logic-Reset (all test logic disabled), Run-Test/Idle (idle between
operations), Select-DR/IR-Scan (choose data or instruction register path),
Capture-DR/IR (load parallel data into shift register), Shift-DR/IR (serial
shift through TDI→TDO), Update-DR/IR (transfer shift register to output
register). The state machine is deterministic -- from any state, TMS=0 and
TMS=1 each lead to exactly one next state. Holding TMS=1 for 5 cycles from
any state guarantees return to Test-Logic-Reset.

### Q11: What are X values in scan testing and why are they problematic?

**A:** X (unknown) values in scan chain responses come from uninitialized
registers, tri-state buses, multi-clock domain captures, and analog interfaces.
In compressed testing, X values are especially problematic because they corrupt
the compactor (XOR network) output -- a single X can mask multiple good
response bits, reducing fault coverage. Mitigation: X-bounding (clamping X
sources via muxes), X-blocking (selectively masking chains per pattern),
X-tolerant compactors, and routing X-heavy FFs to dedicated non-compressed
chains.

### Q12: How do you handle scan testing in a multi-power-domain design?

**A:** Key challenges: (1) Power-gated domains may be OFF during test, breaking
scan chains. (2) Level shifters are needed where scan chains cross voltage
domains. (3) Isolation cells must clamp scan outputs from OFF domains.
Approaches: Force all domains ON for full-chip stuck-at testing; use separate
scan chains per power domain for flexibility; ensure proper isolation and
retention during shift; sequence power domain wake-up through DFT controller.
For at-speed testing, the power management unit must be in a known state.

### Q13: What is the difference between fault coverage and test coverage?

**A:** Fault coverage = detected faults / (total faults - untestable faults).
Test coverage = detected faults / total faults. Fault coverage excludes
proven-untestable faults (tied, blocked, redundant) from the denominator,
giving a more meaningful measure of ATPG effectiveness. Test coverage is more
conservative. Example: 95,000 detected out of 100,000 total, with 2,000
untestable. FC = 95,000/98,000 = 96.9%. TC = 95,000/100,000 = 95.0%.

### Q14: Why is IDDQ testing impractical at advanced nodes?

**A:** At advanced nodes (< 28nm), transistor leakage current increases
exponentially due to thinner gate oxides and shorter channels. At 7nm, normal
leakage can be several uA per gate. For a chip with millions of gates, the
background IDDQ is milliamps to amps, making the additional current from a
single defect (also uA range) undetectable against the noise floor. Delta-IDDQ
(comparing current between vectors) partially helps but adds test time and
requires expensive current-sensing ATE hardware.

### Q15: Explain the PODEM algorithm's advantage over the D-algorithm.

**A:** PODEM restricts all decisions to primary inputs only. When it needs an
internal line to have a certain value, it backtraces to a PI and assigns that
PI. It then forward-simulates (implies) to determine all internal values. This
eliminates the possibility of internal conflicts that plague the D-algorithm
(which makes decisions at internal lines and may discover conflicts only after
extensive computation). PODEM's search space is 2^(number of PIs), while the
D-algorithm's search space includes all internal lines. PODEM is typically
10-50x faster than the D-algorithm on practical circuits.

### Q16: What is the checkpoint theorem?

**A:** The checkpoint theorem states that in a combinational circuit, it is
sufficient to test all faults at **checkpoints** -- which are primary inputs
and fanout branch points -- to guarantee detection of all detectable single
stuck-at faults. Faults on internal single-fanout stems are dominated by
faults at their driving checkpoints. This provides a systematic basis for
fault collapsing: instead of 2N faults (N = lines), we only need to consider
faults at checkpoint locations, typically reducing the fault list by 50%+.

### Q17: How does boundary scan enable board-level testing?

**A:** IEEE 1149.1 boundary scan places shift-register cells (BSCs) at every
I/O pin. Multiple JTAG-compliant chips on a PCB are daisy-chained via
TDI→TDO. Using EXTEST, we can drive known values from one chip's output BSCs
and capture them at another chip's input BSCs, testing every PCB trace for
opens, shorts, and stuck-at faults without physical probe access. This is
especially critical for BGA packages where pins are not probe-accessible.
SAMPLE/PRELOAD allows observing functional I/O values non-intrusively.

### Q18: What determines the number of scan chains in a design?

**A:** The number of scan chains is determined by: (1) Available tester channels
(ATE pins allocated for scan). (2) With compression, internal chain count is
decoupled from pin count -- typically 100-1000 internal chains mapped to 4-16
external channels. (3) Test time budget: more chains = shorter shift time.
(4) Shift power: more chains = more parallel toggling = higher shift power.
(5) Routing congestion: more chains need more scan routing resources.
(6) Chain length balancing: all chains should be within 1 FF of each other
in length for efficient compression.

### Q19: What is the purpose of the BYPASS instruction in JTAG?

**A:** BYPASS loads a single-bit (1-bit) shift register between TDI and TDO.
In a multi-chip JTAG chain, if you only need to access one chip, other chips
are put in BYPASS mode. This shortens the total chain length dramatically.
Without BYPASS, shifting through all chips' boundary scan registers (possibly
thousands of bits each) would be prohibitively slow. With BYPASS, non-targeted
chips add only 1 bit each to the chain, plus 1 clock cycle of latency.

### Q20: How do you reduce test power during scan shift?

**A:** (1) Low-power scan fill: ATPG fills don't-care bits to minimize
adjacent-bit transitions, reducing toggle rate from ~50% to ~10-20%.
(2) Reduce shift frequency or voltage during shift (capture still at speed).
(3) Clock-gating subsets of scan chains so not all toggle simultaneously.
(4) Staggered shift clocking: alternate which chains shift on each cycle.
(5) On-chip power analysis: DFT tools estimate per-pattern power and split
high-power patterns into multiple lower-power patterns if needed.

### Q21: What is retargeting in the context of test compression?

**A:** Retargeting is the process where ATPG determines the external test data
(bits sent from the tester through the decompressor) needed to produce a
specific internal scan chain pattern. Since the decompressor is a deterministic
mapping from external bits to internal chain bits, the ATPG tool must solve
this mapping (typically a system of linear equations over GF(2)) for each
pattern. If a required internal pattern cannot be produced by any external
input (over-constrained), the pattern is split or dropped.

### Q22: Explain the difference between physically exclusive and logically exclusive clock groups.

**A:** In set_clock_groups:
- **Physically exclusive** (-physically_exclusive): Clocks that can NEVER
  coexist on silicon -- e.g., two mux-selected clocks where only one
  propagates at a time. The tool assumes no interaction at all.
- **Logically exclusive** (-logically_exclusive): Clocks that coexist on
  silicon but are functionally mutually exclusive (e.g., test clock vs
  functional clock). The tool allows them to share physical resources but
  doesn't time paths between them.
- **Asynchronous** (-asynchronous): Both clocks exist simultaneously but
  have no phase relationship. The tool doesn't time crossing paths but
  recognizes that both clock trees must be built and may interact physically.

The distinction matters for CTS and power analysis: physically exclusive
clocks can share clock tree resources; the others cannot.

---

## Physical Verification — DRC, LVS, Antenna, and DFM Signoff

*From [Physical_Verification_DRC_LVS.md](../06_Signoff/03_Physical_Verification_DRC_LVS.md)*

**Q: DRC passes but the chip is dead — what check did you skip?** Almost certainly **LVS**: DRC only proves the geometry is legal, not that it implements the right circuit. A poly-to-metal mis-connection or a power/ground short can be perfectly DRC-clean and electrically fatal. LVS (extract + compare to netlist) is what catches it.

**Q: What is the antenna effect and how is it fixed?** A long single-layer metal connected to a thin gate collects plasma charge during fab; if the charge/gate-area ratio exceeds the limit, the gate oxide is damaged. Fix by adding antenna diodes (bleed charge) or jumping the route to another layer so no single segment is too large relative to the gate.

**Q: Why add dummy metal fill if the design already passes DRC?** For **CMP planarity** and yield: chemical-mechanical polishing dishes/erodes low-density regions, changing thickness and hurting yield and timing. Density-balancing fill (and double vias, hotspot fixes) is DFM — making a DRC-clean layout actually high-yielding.

---

## Static Timing Analysis (STA) -- The Complete Interview Bible

*From [STA.md](../06_Signoff/01_STA.md)*

### Q1: Explain NLDM vs CCS delay models. When do you need CCS?

**A:** NLDM uses a 2D lookup table (input slew x output load) to store delay and output slew as single numbers. It models the output load as a lumped capacitance. CCS models the cell as a time-varying current source, capturing the full output current waveform. CCS is needed at 16nm and below because: (1) the output waveform shape is distorted by Miller capacitance and multi-stage driver turn-on, which affects downstream gate delay; (2) the interconnect RC network means different fanout pins see different effective delays; (3) NLDM's lumped-C assumption introduces 10-15% error at advanced nodes, while CCS achieves 2-3% accuracy. The trade-off is CCS libraries are 4-10x larger and STA runs 2-4x slower.

### Q2: What is timing arc unateness? Give examples.

**A:** Unateness describes the relationship between input and output transitions. **Positive unate**: output moves in the same direction as input (AND, OR, buffer). A rise on input causes a rise on output. **Negative unate**: output moves opposite to input (NAND, NOR, inverter). A rise on input causes a fall on output. **Non-unate**: output direction depends on other inputs (XOR, MUX). For XOR, when B=0, A rise causes Y rise; when B=1, A rise causes Y fall. Non-unate arcs force the tool to evaluate both rise/fall combinations, increasing runtime. Unateness is declared in the .lib timing arc as `timing_sense: positive_unate | negative_unate | non_unate`.

### Q3: Derive the setup constraint from first principles. What happens physically when setup is violated?

**A:** Setup time comes from the master-slave FF structure. The master latch (transparent when CLK=0) must have its internal node M settle to a valid logic level before CLK rises and closes TG1. Setup time = the minimum advance time needed for M to reach a voltage that the cross-coupled inverter pair can regenerate to a full rail. If D changes too close to the clock edge, M is caught at an intermediate voltage when TG1 closes. The regenerative latch tries to resolve this metastable voltage, but resolution time is t_resolve = tau * ln(Vdd/2 / |V_meta - Vdd/2|), where tau is the latch time constant. If resolution completes before TG2 opens, we get increased Tc2q but correct data. If not, Q goes metastable and may violate the setup requirement of the next stage, causing a chain of failures.

### Q4: Walk through a PrimeTime timing report. What is each field?

**A:** A PrimeTime setup report shows: (1) **Startpoint**: the launch FF (data source); (2) **Endpoint**: the capture FF (data destination); (3) **Launch clock path**: clock source edge + clock network delay to launch FF; (4) **Tc2q**: propagation delay from CK to Q of launch FF; (5) **Data path**: each combinational cell's delay along the critical path; (6) **Data arrival time**: sum of launch clock + Tc2q + data path; (7) **Capture clock path**: capture clock edge (at next period) + clock network delay to capture FF; (8) **CRPR**: reconvergence pessimism removed from common clock path; (9) **Library setup time**: subtracted from capture clock arrival; (10) **Data required time**: capture clock arrival - CRPR - setup; (11) **Slack**: required - arrival. Positive = MET, negative = VIOLATED. For hold, the capture edge is at t=0 (same edge), delays are min-path, and slack = arrival - required.

### Q5: Explain CRPR with an example. Why does it matter?

**A:** CRPR (Clock Reconvergence Pessimism Removal) corrects the artificial pessimism when OCV derates the shared clock path differently for launch and capture. Example: launch and capture FFs share a common clock buffer B with 100ps delay. For setup, OCV derates launch (early, 95ps) and capture (late, 105ps). But B cannot be both 95ps and 105ps simultaneously -- it's one physical cell. CRPR removes this 10ps difference from the analysis. Without CRPR, the tool computes 10ps of false skew that doesn't exist. CRPR typically recovers 5-50ps of slack per path. It's automatically computed by PrimeTime when `timing_remove_clock_reconvergence_pessimism` is enabled. Larger shared clock path segments yield larger CRPR benefits.

### Q6: Compare OCV, AOCV, and POCV with numbers.

**A:** Consider a path with 30 cells, each with 50ps nominal delay (1500ps total). **Flat OCV** at 5%: late derate = 1575ps, adding 75ps pessimism. **AOCV** with depth-dependent table (depth 30, derate 1.02): late derate = 1530ps, adding 30ps pessimism. **POCV** with sigma=3ps per cell: path sigma = 3 * sqrt(30) = 16.4ps, 3-sigma delay = 1500 + 49.2ps. POCV saves 26ps over flat OCV. The savings increase with path depth. For a 5-cell path, POCV is similar to OCV because sqrt(5) ~= 2.2 doesn't buy much averaging. Modern signoff at 7nm uses POCV; AOCV is common at 16nm; flat OCV is obsolete for advanced nodes.

### Q7: How do multi-cycle paths work? Why must you set the hold MCP?

**A:** MCP tells STA that data takes N cycles to propagate. `set_multicycle_path N -setup` moves the setup check to the Nth capture edge, giving N * Tperiod budget. Without the hold MCP, the hold check remains at the launch edge (t=0), which is overly conservative -- data doesn't need to be stable after t=0, it needs to be stable after the (N-1)th edge. Setting `set_multicycle_path (N-1) -hold` moves the hold check to (N-1) * Tperiod. If you forget this, the tool inserts massive hold buffers between t=0 and the data arrival around N*Tperiod, which wastes area/power and can degrade setup. Rule: always pair setup MCP=N with hold MCP=N-1.

### Q8: What is crosstalk delta delay and how does PrimeTime SI handle it?

**A:** Crosstalk delta delay is the additional delay caused by capacitive coupling to switching aggressors. When an aggressor switches opposite to the victim, the coupling capacitance effectively increases (Miller effect), increasing the victim's delay. PrimeTime SI computes timing windows for all signals, identifies aggressors that switch during the victim's transition window, calculates the delta delay based on coupling capacitance and relative slew rates, and adds it to the path delay. It also performs glitch analysis to check if quiescent victims receive voltage bumps large enough to be captured as logic transitions. SI analysis can add 5-15% to path delay at advanced nodes, making it mandatory for signoff.

### Q9: Walk through a timing ECO flow. What fixes are available?

**A:** After signoff STA shows violations: (1) Identify the violating paths from the timing report; (2) For setup violations: upsize cells (X1->X2), swap Vt (SVT->LVT), add buffers to reduce fanout, restructure logic; (3) For hold violations: insert delay buffers on short paths, swap to HVT; (4) Run incremental STA on affected paths to verify fix; (5) Check that fixes don't break other paths (setup fix may degrade hold, hold fix may degrade setup); (6) Verify physical DRC (max_transition, max_cap, min_spacing); (7) For metal-only ECO (cheapest), use spare cells of the same size/type. For functional ECO, may need to replace filler cells. Always prefer the minimum perturbation that fixes the violation.

### Q10: Explain the complete SDC constraint strategy for a block with 3 clocks: 500MHz core, 200MHz bus, and 33MHz JTAG.

**A:** (1) Define clocks: `create_clock -period 2.0` for core, `-period 5.0` for bus, `-period 30.0` for JTAG. (2) Generated clocks for any PLLs or dividers. (3) `set_clock_groups -asynchronous` between all three if they are truly async. If core and bus are from the same PLL, they are synchronous -- use normal timing or set_multicycle. (4) `set_clock_uncertainty` for each: core needs tight uncertainty (~100-200ps pre-CTS), JTAG can be loose (~500ps). (5) I/O delays on all ports referencing their respective clocks. (6) `set_false_path -from [get_ports TRST_N]` for async JTAG reset. (7) `set_multicycle_path` for any known multi-cycle operations. (8) `set_max_transition/capacitance` for signal integrity. (9) `set_input_transition` on clock ports to model the external clock driver.

### Q11: What is temperature inversion and how does it affect MCMM signoff?

**A:** At advanced nodes (< 28nm, especially FinFET), reducing temperature increases Vth (because the thermal voltage kT/q decreases, steepening the sub-threshold slope and effectively raising Vth). At low supply voltages near Vth, this Vth increase dominates over the mobility improvement, making cells SLOWER at cold temperatures. This means the traditional "slow = hot, fast = cold" mapping breaks. For setup, you must check both SS_hot and SS_cold corners. For hold, check both FF_hot and FF_cold. This can double the number of signoff corners from 2 to 4 for timing.

### Q12: How does clock gating affect STA?

**A:** Clock gating introduces setup and hold checks on the ICG enable signal. The enable must meet setup at the ICG's latch (before the clock falls, for a negative-latch ICG) and hold (after the clock falls). These are called **clock gating checks** and appear as a separate section in the timing report. If the enable fails setup, the gated clock may produce a glitch (partial pulse). If it fails hold, the enable may propagate to the wrong clock cycle. The gated clock then drives downstream FFs, and its timing must be analyzed as a generated clock with the ICG's insertion delay added to the clock tree latency.

### Q13: What is a virtual clock and when do you use it?

**A:** A virtual clock has no physical port -- it's defined with `create_clock -name vclk -period 5.0` without a `get_ports` argument. It's used as a timing reference for I/O constraints when the external clock is not directly available as a pin. For example, if an SPI interface is clocked by an external SPI master clock that doesn't enter your block, you create a virtual clock to model it, then reference it in `set_input_delay -clock vclk`. The tool uses the virtual clock's period and waveform for I/O timing checks but doesn't build a clock tree for it.

### Q14: How do you handle clock MUX selection in STA?

**A:** A clock MUX selects between two clock sources. You define a generated clock on the MUX output for each possible source using `-add` to allow both clocks to coexist. Then use `set_clock_groups -physically_exclusive` to tell STA that only one clock drives the output at a time. Without `-physically_exclusive`, the tool would analyze cross-clock paths between the two MUX inputs, which is impossible. If the MUX is glitch-free (designed with ICG-like circuitry), annotate accordingly. If not, ensure the MUX is only switched during reset or idle periods.

### Q15: What is the difference between set_false_path and set_clock_groups -asynchronous?

**A:** `set_false_path -from clk_a -to clk_b` removes timing checks in ONE direction. You need two commands for bidirectional. `set_clock_groups -asynchronous -group {clk_a} -group {clk_b}` removes timing checks in BOTH directions. Also, `set_clock_groups` applies to all clocks in each group -- if clk_a has generated clocks, they are automatically included. With `set_false_path`, you must list each clock explicitly. `set_clock_groups` is cleaner and less error-prone for clock domain crossings. Additionally, `set_clock_groups` prevents the tool from computing inter-clock uncertainty, which `set_false_path` does not.

### Q16: What are max_transition and max_capacitance constraints? Why do they matter?

**A:** `set_max_transition` limits the slew (rise/fall time) at any pin. Excessive slew causes: increased gate delay (NLDM tables extrapolate poorly), increased short-circuit power (PMOS/NMOS both on longer), potential signal integrity issues (slow edges couple more crosstalk). Typical limits: 200-300ps for data, 100-200ps for clocks. `set_max_capacitance` limits the load on any driver pin. Excessive load causes: slow slew (violates max_transition), increased cell delay, potential reliability issues (electromigration on the driver's output transistors). Both are DRC checks that must be clean for signoff.

### Q17: Explain the timing impact of wire delay vs gate delay at different nodes.

**A:** At 180nm, gate delay dominated (70% gate, 30% wire). At 28nm, the ratio shifted to roughly 50/50. At 7nm, wire delay can dominate for long nets (40% gate, 60% wire) because: (1) wire resistance increases as cross-section shrinks (R ~ 1/Area); (2) gate delay decreased dramatically with scaling; (3) back-end-of-line (BEOL) scaling lagged front-end-of-line (FEOL) scaling. This is why physical design (placement, routing, wire optimization) is increasingly critical for timing closure. Techniques like repeater insertion, layer assignment (use thick upper metals for long nets), and aggressive placement optimization are essential at advanced nodes.

### Q18: How do you verify that your SDC constraints are correct?

**A:** (1) Check for unconstrained paths: `report_timing -unconstrained` should show zero paths. (2) Check for over-constrained paths: review false_path and multicycle_path lists -- each should have a design justification. (3) Check clock waveforms: `report_clocks` should show correct period, waveform, and source. (4) Check I/O budgets: input_delay + internal_delay + output_delay should sum to less than the clock period. (5) Check for conflicting constraints: a path cannot be both false_path and multicycle_path. (6) Run formal equivalence between SDC and the design spec. (7) Cross-check inter-clock relationships: are clocks that should be synchronous treated as synchronous?

### Q19: What is min pulse width check and why is it important?

**A:** Min pulse width ensures the clock pulse (high or low phase) is wide enough for the FF to function correctly. The FF needs minimum time to: (1) fully open the transmission gate, (2) charge/discharge the internal latch node, (3) allow the master/slave handoff. If the pulse is too narrow, the FF may not latch data correctly. This is critical for generated clocks (divided or multiplied) where the effective pulse width may be very short, and for clock gating where the AND gate can narrow the pulse. Typical min pulse width: 40-80% of the cell's Tc2q.

### Q20: Explain how SPEF quality affects timing accuracy.

**A:** SPEF contains extracted parasitic R and C for every net. Quality depends on: (1) **Extraction mode**: detailed (most accurate, models every segment), lumped (fast, less accurate), or coupled (includes coupling caps for SI); (2) **Corner calibration**: extraction must match the PVT corner being analyzed (Cmax for setup, Cmin for hold); (3) **Reduction**: how the RC network is simplified -- too much reduction loses accuracy, too little increases runtime. A 1% error in parasitic extraction can translate to 5-10% error in wire delay for long nets. For signoff, use post-route detailed extraction with corner-specific tech files, and verify extraction accuracy by comparing to a golden SPICE simulation on sample paths.

### Q21: What is useful skew and how does it trade off between paths?

**A:** Useful skew intentionally adjusts clock arrival times at specific FFs to help timing. If path A->B has -50ps setup slack, delaying B's clock by 50ps fixes it. But all other paths ending at B lose 50ps of setup slack, and all paths starting at B gain 50ps of hold margin but lose 50ps of setup-on-next-stage margin. The CTS tool formulates this as a linear programming problem: maximize minimum slack across all paths, subject to the constraint that skew at any FF is the sum of delays through the clock tree (which must be physically realizable with buffers and wire). The total "skew budget" is limited by the tightest hold constraint in the system.

### Q22: How do you handle timing for a path that goes off-chip and comes back (e.g., through an external SRAM)?

**A:** Model it with I/O constraints: (1) `set_output_delay` on the address/data outputs representing the board trace delay + SRAM setup time; (2) `set_input_delay` on the data inputs representing the board trace delay + SRAM access time + Tc2q of SRAM output register; (3) The internal data path is from the output port back to the input port, constrained by `set_max_delay` or through the clock period. For a synchronous SRAM with 2-cycle latency, you might need a multicycle path on the internal feedback path. Always include board-level PCB trace delay (estimate 150ps/inch for FR4) in the I/O delay budgets.

### Q23: What is the timing impact of FinFET self-heating?

**A:** FinFET transistors are thermally isolated by the oxide surrounding the fin. Under continuous switching, the channel temperature rises 10-30C above ambient (self-heating). This increases delay because higher temperature increases resistance and (at advanced nodes) threshold voltage. STA libraries at 7nm and below include self-heating derating factors. The delay increase is workload-dependent: high-activity paths heat up more. Tools model this either through adjusted timing libraries (already derated for typical self-heating) or through separate self-heating analysis that identifies hot spots and applies per-instance derating.

### Q24: Explain the concept of "borrowing" in latch-based timing.

**A:** In a latch-based design, data can arrive after the opening edge of the latch and still be captured, as long as it arrives before the closing edge. The time between the opening and closing edge is the "transparency window." Time borrowing allows a slow stage to borrow time from the next stage's budget. Example: with a 50% duty cycle 2ns clock, stage 1 can use up to 2ns (1ns from its half-cycle + 1ns borrowed from stage 2's transparency window). Stage 2 then has less time (only the remaining portion of its half-cycle). This is why latch-based designs can achieve higher frequency than FF-based designs -- the clock period is not rigidly partitioned. STA for latches uses time borrowing checks instead of setup/hold.

### Q25: What is the signoff checklist for taping out a design?

**A:** The timing signoff checklist:
1. All setup/hold/recovery/removal slack >= 0 at ALL MCMM scenarios
2. Max transition clean (no pin exceeds transition limit)
3. Max capacitance clean (no driver overloaded)
4. Min pulse width clean (all clock pulses wide enough)
5. Clock gating checks clean
6. No crosstalk noise failures (SI clean)
7. No unconstrained paths (complete SDC coverage)
8. No missing clock definitions or false clock relationships
9. CRPR enabled and correctly computed
10. POCV/AOCV tables validated against silicon data
11. Parasitics extracted at all signoff corners with calibrated tech files
12. IR drop analysis shows no timing-significant voltage droop
13. Electromigration clean on all signal and clock nets
14. DFT timing clean (scan shift and capture modes)
15. Formal verification of SDC constraints against design intent

