# Power Analysis and Signoff for Digital IC / ASIC Design

## 1. Power Analysis Methodology Overview

Power analysis occurs at every stage of the design flow. Each stage trades speed for accuracy:

```
  RTL Power Estimation (~20-30% error)
      |  Tools: Synopsys Joules, Ansys PowerArtist, Cadence Genus (power mode)
      |  Input: RTL + SAIF + technology characterization data
      |  Speed: minutes to hours
      |  Use: early budgeting, architecture exploration
      v
  Gate-Level Power Analysis (~10-15% error)
      |  Tools: Synopsys PrimeTime PX, Cadence Voltus
      |  Input: Netlist + SAIF/VCD + estimated SPEF (pre-route)
      |  Speed: hours
      |  Use: post-synthesis optimization, power closure
      v
  Post-Layout Power Analysis (~3-5% error)
      |  Tools: PrimeTime PX, Voltus, ANSYS RedHawk
      |  Input: Netlist + extracted SPEF + SAIF/VCD + DEF
      |  Speed: hours to days
      |  Use: signoff, final power numbers
      v
  Power Signoff
      |  Final check against budget, IR drop, EM limits
      v
  Tapeout
```

### 1.1 Why Accuracy Varies

| Stage          | Missing Information                          | Impact on Accuracy         |
|----------------|----------------------------------------------|----------------------------|
| RTL            | No gate-level netlist, no wire caps          | Miss glitch power, wire C  |
| Post-synthesis | Wire cap estimated from fanout (wireload)    | Wire C can be 30-50% off   |
| Post-layout    | Real extracted parasitics (SPEF)             | Best accuracy available    |

---

## 2. Activity Annotation -- Deep Dive

### 2.1 SAIF Format (Switching Activity Interchange Format)

SAIF captures per-net switching statistics over a simulation window. It records toggle
count, time at logic-0, and time at logic-1 for every annotated net.

**Complete SAIF file example:**
```
(SAIF
  (SAIFILE
    (DIRECTION "backward")
    (DESIGN "soc_top")
    (DATE "2026-03-15 14:30:00")
    (VENDOR "Synopsys")
    (PROGRAM_NAME "VCS")
    (VERSION "2025.06-SP1")
    (DIVIDER /)
    (TIMESCALE 1 ps)
    (DURATION 2000000)

    (INSTANCE soc_top
      (NET
        (clk
          (T0 1000000) (T1 1000000) (TC 4000) (IG 0)
          ;; clk: 50% duty cycle, 4000 transitions in 2000000 ps (2 us)
          ;; -> 1 GHz clock (TC=4000 edges / 2 ns period = 2000 cycles / 2 us = 1 GHz)
          ;; SP = 0.5, TR = 4000/2000000 = 0.002 transitions/ps = 2e9/s
        )
        (reset_n
          (T0 200000) (T1 1800000) (TC 2) (IG 0)
          ;; reset_n: held low for 200ns, then high. 2 transitions total.
          ;; SP = 0.9, TR = 1e-6 transitions/ps (essentially static)
        )
      )
      (INSTANCE cpu_core
        (NET
          (alu_result\[0\]
            (T0 850000) (T1 1150000) (TC 2800) (IG 4)
            ;; Bit 0 of ALU result: SP=0.575, TC=2800
            ;; IG=4 means 4 transitions involved X states (during reset)
          )
          (alu_result\[31\]
            (T0 1600000) (T1 400000) (TC 600) (IG 4)
            ;; MSB toggles less (sign bit, mostly 0 for positive numbers)
            ;; SP=0.2, TC=600
          )
          (pc\[0\]
            (T0 1000000) (T1 1000000) (TC 3000) (IG 2)
            ;; PC bit 0 toggles frequently (every instruction)
          )
          (pc\[15\]
            (T0 1950000) (T1 50000) (TC 20) (IG 2)
            ;; PC bit 15 toggles rarely (high address bit)
          )
        )
        (INSTANCE reg_file
          (NET
            (data_out\[0\]
              (T0 900000) (T1 1100000) (TC 1500) (IG 0)
            )
          )
        )
      )
    )
  )
)
```

**Derived metrics from SAIF:**
```
For net alu_result[0]:
  Static Probability (SP) = T1 / DURATION = 1150000 / 2000000 = 0.575
  Toggle Rate (TR)        = TC / DURATION = 2800 / 2000000 = 0.0014 transitions/ps
  Toggle Count per cycle  = TC / num_clk_cycles = 2800 / 2000 = 1.4 transitions/cycle
  Switching Activity (SA) = TC / (2 * num_clk_cycles) = 2800 / 4000 = 0.7
      (SA > 0.5 means more than one toggle per clock period on average -- possible
       if there are glitches within a cycle)
```

### 2.2 Forward vs Backward SAIF Annotation

**Forward annotation:**
```
1. Simulate RTL
2. Dump SAIF at RTL net names
3. During power analysis on gate-level netlist:
   - Tool maps RTL net names to gate-level net names
   - Mapping is imperfect (synthesis may rename/restructure)
   - Unmapped nets use default activity -> inaccuracy
   
Advantage: RTL simulation is fast (10-100x faster than gate-level)
Disadvantage: naming mismatch, missing glitch activity
```

**Backward annotation:**
```
1. Run power tool on gate-level netlist
2. Tool generates a SAIF template with ALL gate-level net names
3. Simulate gate-level netlist, dump SAIF matching the template
4. Re-run power tool with filled-in SAIF

Advantage: perfect name matching, captures glitch activity
Disadvantage: gate-level simulation is slow, SAIF file is large
```

**Practical recommendation:**
- Use forward annotation for early estimation (RTL development phase)
- Use backward annotation for signoff (gate-level SAIF for final power numbers)
- For very large designs: hybrid (gate-level SAIF on critical blocks, vectorless for rest)

### 2.3 VCD (Value Change Dump)

VCD records EVERY signal transition with a timestamp. This enables time-based power analysis.

```
$date
   2026-03-15 14:30:00
$end
$version VCS 2025.06 $end
$timescale 1ps $end
$scope module soc_top $end
$var wire 1 ! clk $end
$var wire 1 " reset_n $end
$var wire 32 # data_bus [31:0] $end
$upscope $end
$enddefinitions $end

#0
0!
0"
b00000000000000000000000000000000 #

#500
1!

#1000
0!

#1500
1!
1"

#2000
0!
b00000000000000001010101010101010 #

#2500
1!
```

**VCD file size problem:**
```
For a design with 10M nets, toggling at average 0.1 transitions/ns, over 10us:
  Total transitions = 10M * 0.1 * 10,000 = 10 billion transitions
  Each transition: ~20 bytes (timestamp + signal ID + value)
  VCD size = 10e9 * 20 = 200 GB

This is impractical. Solutions:
  1. FSDB (Synopsys): compressed binary format, 5-10x smaller
  2. Selective dumping: only dump signals in the block under analysis
  3. Shorter simulation window (but must be representative)
  4. SAIF instead of VCD (for average power, not time-based)
```

### 2.4 VCD vs FSDB vs SAIF Comparison

| Feature            | VCD                  | FSDB                 | SAIF                 |
|--------------------|----------------------|----------------------|----------------------|
| Format             | Text (ASCII)         | Binary (compressed)  | Text (hierarchical)  |
| File size          | Huge (10-100 GB)     | Large (1-20 GB)      | Small (10-100 MB)    |
| Time resolution    | Per-transition       | Per-transition       | Aggregated stats     |
| Time-based power   | Yes                  | Yes                  | No (average only)    |
| Peak power         | Yes                  | Yes                  | No                   |
| Glitch capture     | Yes (gate-level)     | Yes (gate-level)     | Yes (via TC count)   |
| Standard           | IEEE 1364            | Proprietary          | Synopsys (de facto)  |
| Random access      | No (sequential)      | Yes                  | N/A                  |

### 2.5 Vectorless Power Estimation

When simulation is not possible (too early in design, too expensive, or workload unknown):

**How vectorless estimation works:**
```
1. Assign default toggle rate (TR) and static probability (SP) to primary inputs
   Typical defaults: SP = 0.5, TR = 0.1 (one transition every 10 clock cycles)

2. Propagate through the netlist:
   For an AND gate with inputs A (SP_A, TR_A) and B (SP_B, TR_B):
     SP_out = SP_A * SP_B
     TR_out = SP_A * TR_B + SP_B * TR_A  (approximate, assumes independence)
   
   For an inverter:
     SP_out = 1 - SP_in
     TR_out = TR_in

3. Calculate power per cell using the propagated activity

4. Sum over all cells
```

**When vectorless is ACCURATE:**
- Clock tree power (activity = 1.0 per cycle, known exactly)
- Random data paths (when data really is random)
- Relative comparisons between design alternatives (if same assumptions)
- Worst-case estimation (set high TR for pessimistic analysis)

**When vectorless is INACCURATE:**
- Modules with enable signals (vectorless may not model enables correctly -> overestimates
  by assuming logic is always active)
- Data-dependent activity (multiplier toggling depends on operand correlation)
- Deeply gated structures (vectorless doesn't know the enable is usually off)
- State machines with dominant idle state

**Hybrid approach (best practice):**
```tcl
# In PrimeTime PX:
# Annotate SAIF on critical blocks (CPU core, memory controller)
read_saif critical_blocks.saif -strip_path tb/dut

# Set vectorless defaults for remaining unannotated logic
set_switching_activity -static_probability 0.5 \
    -toggle_rate 0.1 \
    -base_clock clk \
    [get_nets -hier * -filter "is_annotated == false"]

# Check annotation coverage
report_switching_activity -unannotated > unannotated.rpt
# Target: >80% annotation coverage for signoff
```

---

## 3. IR Drop Analysis -- Deep Dive

### 3.1 What Is IR Drop?

Every wire in the power grid has resistance. When current flows through this resistance,
voltage drops (V = I*R), reducing the supply voltage seen by standard cells:

```
  Package Pin (Vdd = 0.9V)
      |
    [R_package] ~5-20 mOhm
      |
  Chip Pad (0.895V)
      |
    [R_pad + R_bump] ~10-50 mOhm
      |
  Top Metal Ring (0.890V)
      |
    [R_stripe_M9] ~0.1-1 ohm/mm
      |
    [R_via_array]
      |
    [R_stripe_M7]
      |
    [R_via_array]
      |
    [R_stripe_M5]
      |  ... more metal layers ...
      |
  Standard Cell Rail (0.855V)
      |
    [R_local] ~1-10 ohm
      |
  Transistor Source (0.850V)

  Total IR drop: 0.9 - 0.85 = 50 mV = 5.6% of Vdd
```

### 3.2 Static IR Drop

Static IR drop assumes DC (steady-state) current. It is a pure resistive network analysis:

**The resistor mesh model:**
```
  VDD ring (top)
   |        |        |        |
  [Rv]     [Rv]     [Rv]     [Rv]      (via resistance to power stripe)
   |        |        |        |
  ---[Rh]---+---[Rh]---+---[Rh]---     M7 power stripe (horizontal)
   |        |        |        |
  [Rv]     [Rv]     [Rv]     [Rv]      (via to next layer)
   |        |        |        |
  ---[Rh]---+---[Rh]---+---[Rh]---     M5 power stripe (horizontal)
   |        |        |        |
  [Rv]     [Rv]     [Rv]     [Rv]
   |        |        |        |
  ---[Rh]---+---[Rh]---+---[Rh]---     M1 cell rail (horizontal)
   |        |        |        |
  [I1]     [I2]     [I3]     [I4]      (current sinks = standard cells)

Each current sink I_i represents the current drawn by cells in that region.
```

**Static IR drop analysis:**
- Model the power grid as a resistor network
- Apply current sources at each cell location (from average power analysis)
- Solve the linear system V = R^(-1) * I using a sparse matrix solver
- Report voltage at every node

**Typical targets:**
```
Static IR drop < 5% of Vdd (some companies: < 3%)
For Vdd = 0.9V: maximum IR drop = 45 mV (5%) or 27 mV (3%)
```

### 3.3 Dynamic IR Drop

Dynamic IR drop is MUCH worse than static because of transient current demands:

**Why dynamic IR drop peaks at clock edges:**
```
At a clock edge, ALL flip-flops in the design sample simultaneously.
This creates a massive, near-simultaneous current demand:

  I_peak = N_switching_FFs * C_FF * Vdd * f_local

  For a region with 10K FFs, 30% switching:
  I_peak = 3000 * 20fF * 0.9V / (0.5ns half-period)
         = 3000 * 20e-15 * 0.9 / 0.5e-9
         = 3000 * 36e-6
         = 108 mA locally!

This transient current through grid resistance AND inductance:
  V_drop = I*R + L*(dI/dt)

The Ldi/dt component can be 2-3x worse than the IR component.
```

**Dynamic IR drop characteristics:**
```
  Time -->
  
  Vdd_local
  0.90|```````\     ______/``````\     ______/````
  0.85|        \   /      |       \   /
  0.80|         \_/       |        \_/
  0.75|                   |
      +------|------|-----|------|------|-----
             CLK   CLK   CLK   CLK   CLK
             edge  edge  edge  edge  edge
  
  Peak dynamic IR drop of 150mV (16.7%) at clock edges
  vs static IR drop of maybe 30-40mV (3-4%)
```

**Dynamic IR drop targets:**
```
Dynamic IR drop < 10% of Vdd (typical)
Some high-performance designs: < 8%
For Vdd = 0.9V: maximum dynamic drop = 90 mV (10%)

If dynamic IR drop exceeds spec:
  - Timing failures (cells see lower Vdd -> slower -> setup violations)
  - Functional failures at extreme drop (below noise margin)
  - Reliability issues (oxide stress, hot carrier injection from recovery overshoot)
```

### 3.4 Decoupling Capacitors (Decap)

Decaps provide local charge storage to supply transient current demand, reducing
dynamic IR drop:

```
Frequency domain analysis of decap effectiveness:

  On-die MOS-cap (gate capacitance of NMOS):
    - Effective up to ~1-5 GHz
    - Provides ~1-5 nF/mm^2 in modern processes
    - Placed in whitespace (filler cells with decap)
    
  Explicit decap cells in standard cell library:
    - 2-10 fF per cell (varies by size)
    - Placed where IR drop is worst
    - Area cost: typically 5-10% of chip area dedicated to decap
    
  Package decap:
    - Surface mount capacitors on package substrate
    - 10-100 nF typical
    - Effective up to ~100-500 MHz (limited by package inductance)
    
  PCB decap:
    - Bulk capacitors on the board
    - 1-100 uF
    - Effective up to ~10-50 MHz only
    
  Each level covers a different frequency band:
  
  Frequency:  1Hz    1kHz   1MHz   100MHz  1GHz   10GHz
              |------|------|------|-------|------|
              [PCB decap                  ]
                     [Package decap       ]
                            [On-die decap         ]
                                   [Intrinsic cell cap]
```

**Decap sizing calculation:**
```
Given:
  Transient current demand: I_peak = 200 mA for 200 ps duration
  Allowable voltage droop: delta_V = 50 mV

Required local decap:
  C = I * dt / dV = 200e-3 * 200e-12 / 50e-3 = 800 pF

On-die decap density: ~2 nF/mm^2
Area needed: 800pF / 2nF/mm^2 = 0.4 mm^2

If chip area is 10 mm^2, this is 4% of chip area for this region alone.
```

### 3.5 Electromigration (EM)

EM is the gradual displacement of metal atoms by momentum transfer from conducting
electrons. It causes wire thinning (open circuit) or hillock formation (short circuit).

**Black's Equation:**
```
MTTF = A * J^(-n) * exp(Ea / (k*T))

where:
  MTTF = Mean Time To Failure
  A    = process constant
  J    = current density (A/cm^2)
  n    = current density exponent (typically 1-2, usually 2 for bulk EM)
  Ea   = activation energy (0.7-0.9 eV for Cu with liner)
  k    = Boltzmann constant (8.617e-5 eV/K)
  T    = absolute temperature (K)
```

**EM design rules:**
```
DC EM limit (Jmax_DC): typically 1-3 MA/cm^2 for copper at 105C
AC EM limit (Jmax_AC): typically 2-6 MA/cm^2 (2x DC due to self-healing)

Wire sizing for EM:
  Required wire width = I_rms / (Jmax * t_metal)
  
  Example:
    I_avg = 10 mA through a M5 power stripe
    Jmax_DC = 2 MA/cm^2 = 2e6 A/cm^2
    t_metal = 400 nm (M5 copper thickness)
    
    Required width = 10e-3 / (2e6 * 400e-7) = 10e-3 / 0.08 = 0.125 cm = 1250 um
    
    This means a single narrow stripe cannot carry 10 mA!
    Need multiple parallel stripes or one wide stripe (~1.25 mm).
```

**Temperature dependence of EM:**
```
From Black's equation, MTTF ~ exp(Ea / (kT))

At T1 = 105C (378K): exp(0.7 / (8.617e-5 * 378)) = exp(21.5)
At T2 =  85C (358K): exp(0.7 / (8.617e-5 * 358)) = exp(22.7)

MTTF ratio: exp(22.7) / exp(21.5) = exp(1.2) = 3.3x

85C has 3.3x longer lifetime than 105C for the same current density.
Every 20C increase in temperature roughly halves EM lifetime (for Ea=0.7eV).
```

**EM at advanced nodes:**
```
As wire cross-sections shrink (thinner, narrower):
  - Current density increases for same total current
  - Copper resistivity increases (grain boundary scattering at thin films)
  - EM becomes a tighter constraint
  
Mitigation:
  - Cobalt cap layer (better EM resistance than copper/barrier interface)
  - Ruthenium liner/barrier (thinner, more copper volume)
  - Backside power delivery (move power off the signal layers)
```

### 3.6 Power Grid Design

**Typical power grid structure:**
```
  +============================================================+
  |  VDD Ring (top-level, M10)                                  |
  |  +------------------------------------------------------+  |
  |  | VDD Stripes (M9, horizontal, ~5-10 um pitch)         |  |
  |  |  ===  ===  ===  ===  ===  ===  ===  ===  ===  ===   |  |
  |  | VSS Stripes (M9, horizontal, alternating)            |  |
  |  |  ---  ---  ---  ---  ---  ---  ---  ---  ---  ---   |  |
  |  |                                                      |  |
  |  | VDD Stripes (M8, vertical, ~5-10 um pitch)          |  |
  |  |  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||    |  |
  |  | VSS Stripes (M8, vertical, alternating)              |  |
  |  |  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||  ||    |  |
  |  |                                                      |  |
  |  | Via arrays at every M8/M9 intersection               |  |
  |  |                                                      |  |
  |  | Lower metals (M1-M4): standard cell power rails      |  |
  |  | M1 VDD/VSS rails in every standard cell row          |  |
  |  +------------------------------------------------------+  |
  |  VSS Ring (M10)                                             |
  +============================================================+
```

**Power grid design guidelines:**
```
General rules:
  - Dedicate 5-10% of each metal layer to power routing
  - Top 2-3 metals carry most of the current (thickest, lowest resistance)
  - Alternate VDD/VSS stripes for balanced current distribution
  - Via arrays at every intersection (maximize via count for low resistance)
  - Power rings around the chip periphery (connect to bumps/pads)

Trade-offs:
  Wider stripes:
    + Lower IR drop (lower R)
    + Better EM (lower J for same I)
    - Less routing resource for signals
    - Potential routing congestion
  
  Denser stripe pitch:
    + More uniform voltage distribution
    + Better current sharing
    - More routing resource consumed
    - Diminishing returns (mesh already dense enough)

Typical pitch guidelines (for a medium-performance design):
  M8/M9 (top metals): VDD+VSS stripe every 20-40 um
  M6/M7 (mid metals):  VDD+VSS stripe every 30-60 um
  M1 (cell rail): continuous VDD and VSS in every cell row
```

---

## 4. Power Signoff

### 4.1 Multi-Corner Power Analysis

Power signoff requires analysis at multiple PVT corners:

| Corner              | Temp | Voltage | Process | Purpose                      |
|---------------------|------|---------|---------|------------------------------|
| Typical (TT/0.9V/25C) | 25C  | Nominal | Typical | Average power reporting      |
| Worst Leakage (FF/max V/125C) | 125C | Max     | Fast    | Maximum leakage power        |
| Worst Dynamic (SS/min V/m40C) | -40C | Min     | Slow    | Maximum dynamic power (highest freq at coldest) -- actually this is for timing, not power |
| Signoff Power (TT/0.9V/85C) | 85C  | Nominal | Typical | Realistic operating point    |
| Min Power (SS/min V/m40C) | -40C  | Min     | Slow    | Min power (for regulator design) |

**Why minimum power matters:**
The voltage regulator must handle the LOAD TRANSIENT between minimum and maximum power.
If min power is 100mW and max is 2W, the regulator must handle a 20x load step.
Knowing both extremes helps size the regulator's output capacitors and control bandwidth.

### 4.2 Power Signoff Criteria

| Criterion                | Typical Spec                    | Notes                          |
|--------------------------|---------------------------------|--------------------------------|
| Static IR drop           | < 5% Vdd (45 mV for 0.9V)      | Average current based          |
| Dynamic IR drop          | < 10% Vdd (90 mV for 0.9V)     | Peak transient at clock edges  |
| EM lifetime (DC)         | > 10 years at 105C              | Based on Black's equation      |
| EM lifetime (AC)         | > 10 years at 105C              | AC limit ~2x DC limit          |
| Average power            | Within power budget              | Per-domain and total           |
| Peak power               | Within package/regulator limit   | May need to add decap          |
| Leakage at max temp      | Within thermal envelope          | Check for thermal runaway      |

### 4.3 PrimeTime PX Complete Signoff Flow

```tcl
#================================================================
# PrimeTime PX Power Signoff Flow (Post-Layout)
#================================================================

# 1. Setup
set search_path "/libs/7nm/nldm /design/db /design/spef"
set link_library "* ss_0p72v_m40c.db tt_0p90v_25c.db ff_0p99v_125c.db"

# 2. Read design
read_verilog /design/netlist/soc_top_postroute.v
current_design soc_top
link_design

# 3. Read timing constraints
read_sdc /design/constraints/soc_top.sdc

# 4. Set analysis corner
set_operating_conditions tt_0p90v_85c \
    -library tt_0p90v_85c

# 5. Read post-layout parasitics
read_parasitics /design/spef/soc_top_Cmax.spef.gz

# 6. Annotate switching activity
# Option A: SAIF (for average power)
read_saif /sim/results/soc_top_workload.saif \
    -strip_path testbench/dut

# Option B: VCD (for time-based power)
# read_vcd /sim/results/soc_top_workload.vcd \
#     -strip_path testbench/dut

# 7. Check annotation coverage
report_switching_activity -list_not_annotated > unannotated.rpt
# If coverage < 80%, investigate missing annotations

# 8. Set default activity for unannotated nets
set_switching_activity -toggle_rate 0.1 \
    -static_probability 0.5 \
    -base_clock clk \
    [all_inputs]

# 9. Update power analysis
update_power

# 10. Generate reports
# Hierarchical power breakdown
report_power -hierarchy -levels 3 \
    -nosplit > reports/power_hierarchy.rpt

# Power by cell type
report_power -cell_power -sort_by total \
    -nosplit > reports/power_by_cell.rpt

# Power by net (top switching nets)
report_power -net_power -sort_by switching \
    -limit 100 > reports/power_top_nets.rpt

# Leakage breakdown by Vt
report_power -threshold_voltage_group \
    > reports/power_by_vt.rpt

# Power by power domain (if multi-domain design)
report_power -power_domain > reports/power_by_domain.rpt

# 11. Detailed power for specific instances
report_power -instances {cpu_core gpu_core mem_ctrl} \
    -verbose > reports/power_critical_blocks.rpt

# 12. For time-based analysis (requires VCD):
# set_power_analysis_options -waveform_interval 1ns
# set_power_analysis_options -include peak
# update_power
# report_power -time_based > reports/power_waveform.rpt
# report_power -peak -instances {cpu_core} > reports/peak_power.rpt
```

### 4.4 Voltus / RedHawk IR Drop Analysis Flow

```tcl
#================================================================
# Cadence Voltus Rail Analysis (IR Drop + EM) Flow
#================================================================

# 1. Import design
read_lib /libs/7nm/lib/tt_0p90v_25c.lib
read_lef /libs/7nm/lef/tech.lef
read_lef /libs/7nm/lef/stdcells.lef
read_def /design/def/soc_top.def
read_verilog /design/netlist/soc_top.v

# 2. Read parasitics
read_spef /design/spef/soc_top.spef

# 3. Power grid extraction
# Extract the power/ground network from DEF + technology
set_pg_net_connectivity -auto

# 4. Set power analysis mode
set_rail_analysis_mode \
    -method static \
    -accuracy xd \
    -power_grid_library /libs/7nm/pglib/pglib.cl

# 5. Specify current sources (from power analysis)
# Either read power data from a previous Voltus power run:
read_power_rail_results -power_db /design/power/soc_top_power.db

# Or apply uniform current density:
# set_rail_analysis_domain -name VDD -power_net VDD -ground_net VSS
# set_power_data -format current -scale 1.0

# 6. Run static IR drop analysis
analyze_rail -type net -net VDD
analyze_rail -type net -net VSS

# 7. Report static IR drop
report_rail -type net -net VDD \
    -format detail > reports/ir_drop_vdd.rpt

# 8. Generate IR drop map (visual)
report_rail -type net -net VDD \
    -format map \
    -output_file reports/ir_drop_map.png

# 9. Dynamic IR drop analysis (requires VCD/FSDB)
set_rail_analysis_mode -method dynamic
set_dynamic_rail_analysis \
    -power_format fsdb \
    -power_file /sim/results/soc_top.fsdb

# Specify analysis window (around worst-case clock edge)
set_dynamic_rail_analysis -start_time 100ns -end_time 110ns

analyze_rail -type net -net VDD

report_rail -type dynamic -net VDD \
    -format detail > reports/dynamic_ir_drop.rpt

# 10. EM analysis
analyze_rail -type em -net VDD
analyze_rail -type em -net VSS

report_rail -type em -net VDD \
    -format detail > reports/em_vdd.rpt

# Check for EM violations
report_rail -type em -net VDD \
    -violation_only > reports/em_violations.rpt
```

### 4.5 ANSYS RedHawk Flow (Alternative to Voltus)

```tcl
#================================================================
# ANSYS RedHawk (now Ansys Totem) IR Drop Flow
#================================================================

# Import design
import_design -lef tech.lef stdcells.lef \
    -def soc_top.def \
    -verilog soc_top.v

# Setup technology
setup_technology -metal_stack /tech/metal_stack.tech

# Import power grid
import_power_grid -def soc_top.def

# Setup analysis
setup_analysis \
    -type dynamic \
    -frequency 1e9 \
    -temperature 85

# Apply power (from PrimeTime PX or internal)
import_power_data -format ptpx \
    -file soc_top_power.fsdb

# Run analysis
run_analysis -type all  ;# static + dynamic + EM

# Generate reports and maps
report_static_ir -net VDD -threshold 30mV > static_ir.rpt
report_dynamic_ir -net VDD -threshold 60mV > dynamic_ir.rpt
report_em -net VDD -lifetime 10 > em.rpt

# Power integrity map
generate_map -type dynamic_ir -net VDD -output ir_map.png
```

---

## 5. Power Budgeting for SoC

### 5.1 Top-Down Power Budgeting

Start from the total power constraint and allocate downward:

```
Example: Mobile Application Processor
  Total TDP (Thermal Design Power): 5W at 85C junction temperature
  
  +----------------------------------+-------+--------+
  | Subsystem                        | Budget| % Total|
  +----------------------------------+-------+--------+
  | CPU Cluster (4x A-class cores)   | 2.0 W | 40%    |
  |   Per-core budget                | 0.5 W |        |
  | GPU (8-core)                     | 1.5 W | 30%    |
  | Modem (5G baseband)              | 0.5 W | 10%    |
  | Memory Controller (LPDDR5)       | 0.3 W |  6%    |
  | Display Processor                | 0.2 W |  4%    |
  | IO Subsystem (USB, PCIe, UFS)    | 0.2 W |  4%    |
  | Always-On Domain (PMU, RTC, AON) | 0.05W |  1%    |
  | Interconnect (NoC, bus fabric)   | 0.15W |  3%    |
  | Margin                           | 0.1 W |  2%    |
  +----------------------------------+-------+--------+
  | TOTAL                            | 5.0 W | 100%   |
  +----------------------------------+-------+--------+
```

### 5.2 Bottom-Up Power Verification

After design, sum actual block powers and compare with budget:

```
For each block:
  1. Run gate-level power analysis with representative workload
  2. Get average power (dynamic + leakage)
  3. Get peak power
  4. Report per-domain breakdown

Comparison:
  If actual > budget: need optimization (clock gating, Vt, DVFS, redesign)
  If actual < budget: margin available (or budget was too generous)

Iteration:
  Power budgets are refined at each design milestone:
    - Architecture phase: rough estimates (historical data, scaling)
    - RTL phase: RTL power estimation
    - Post-synthesis: gate-level analysis
    - Post-layout: final signoff numbers
```

### 5.3 Power Modes and Use Cases

```
Mobile SoC Power Modes:

Mode           | CPU   | GPU   | Modem  | IO    | Total  | Use Case
---------------|-------|-------|--------|-------|--------|------------------
Active (Max)   | 2.0W  | 1.5W  | 0.5W   | 0.2W  | 5.0W   | Gaming, benchmark
Active (Typ)   | 0.8W  | 0.5W  | 0.3W   | 0.1W  | 2.0W   | Normal use
Idle           | 0.1W  | Off   | 0.1W   | 0.05W | 0.4W   | Screen on, no task
Light Sleep    | Ret   | Off   | Idle   | Off   | 0.05W  | Screen off
Deep Sleep     | Off   | Off   | Idle   | Off   | 0.01W  | Phone in pocket
Hibernate      | Off   | Off   | Off    | Off   | 0.002W | Power button off

Key: Ret = retention (state preserved), Off = fully power-gated
```

**Battery life calculation:**
```
Battery: 5000 mAh at 3.8V = 19 Wh

Light sleep mode (50 mW):
  Battery life = 19 Wh / 0.05W = 380 hours = ~16 days standby

Active typical (2W):
  Battery life = 19 Wh / 2W = 9.5 hours screen-on time

Active max (5W, gaming):
  Battery life = 19 Wh / 5W = 3.8 hours gaming
```

---

## 6. Thermal Analysis

### 6.1 Junction Temperature

```
Tj = Ta + P * Rth_ja

where:
  Tj     = junction temperature (C)
  Ta     = ambient temperature (C)
  P      = total power dissipation (W)
  Rth_ja = thermal resistance from junction to ambient (C/W)

Rth_ja depends on the package and cooling solution:
  Bare die (no heatsink):    Rth_ja ~ 30-60 C/W
  With heatsink (laptop):    Rth_ja ~ 5-15 C/W
  With fan (desktop):        Rth_ja ~ 2-5 C/W
  Liquid cooling (server):   Rth_ja ~ 0.5-2 C/W
  
Thermal resistance chain:
  Rth_ja = Rth_jc + Rth_cs + Rth_sa
  
  Rth_jc = junction to case (die + package) ~ 0.5-5 C/W
  Rth_cs = case to heatsink (thermal paste/pad) ~ 0.1-1 C/W
  Rth_sa = heatsink to ambient (convection + radiation) ~ 1-30 C/W
```

**Example:**
```
Mobile SoC: 5W, no heatsink (just PCB copper), Rth_ja = 40 C/W, Ta = 45C (in pocket)
  Tj = 45 + 5 * 40 = 245C -> IMPOSSIBLE, chip would burn

This is why mobile chips throttle at high power:
  Maximum Tj = 100C (typical mobile spec)
  Max sustainable power = (100 - 45) / 40 = 1.375W at 45C ambient

The 5W TDP is only achievable for short bursts (transient thermal response
allows brief power spikes before the die heats up).
```

### 6.2 Thermal Runaway

Thermal runaway is a positive feedback loop:

```
  Higher temperature
      |
      v
  More leakage (2x per 10C)
      |
      v
  More power dissipation
      |
      v
  Higher temperature
      |
      v
  More leakage ... (positive feedback!)
```

**Stability analysis:**
```
Power dissipated: P(T) = P_dynamic + P_leakage(T)
  P_leakage(T) = P_leak_25C * 2^((T-25)/10)

Heat removed:    Q(T) = (T - Ta) / Rth_ja

Stable operating point exists where P(T) = Q(T)

Graphically:
  Power
  (W)
   |         .......P(T) (exponential, leakage-dominated)
   |        .
   |       .    /  Q(T) (linear, heat removal)
   |      .   /
   |     .  /
   |    . /     <-- STABLE operating point (intersection)
   |   ./
   |  /
   | /
   +-------|-------|--------> Temperature
          T_stable  T_runaway

If the P(T) curve is STEEPER than Q(T) at the intersection:
  -> UNSTABLE -> thermal runaway -> chip destruction
  
This happens when:
  dP/dT > dQ/dT
  P_leak_25C * 2^((T-25)/10) * ln(2)/10 > 1/Rth_ja
```

**Design implication:** You MUST verify that a stable thermal operating point exists
at the worst-case ambient temperature. If leakage is too high relative to cooling
capability, the chip will thermally run away.

### 6.3 Dynamic Thermal Management (DTM)

Since sustainable power is often much less than peak power, DTM is essential:

**On-chip temperature sensors:**
```
  Ring oscillator-based: frequency decreases with temperature
    - Simple, area-efficient
    - Accuracy: +/- 5-10C
    - Placed at expected hotspots (CPU core, GPU, memory controller)
    
  BJT-based (bandgap reference):
    - V_be decreases linearly with temperature (~-2 mV/C)
    - Compare V_be with reference using ADC
    - Accuracy: +/- 1-3C (better than ring osc)
    - Used in most production SoCs

  Typical placement: 4-8 sensors per chip, at least one per major block
```

**Throttling mechanisms (in order of severity):**
```
Level 0 (Proactive): DVFS governor reduces frequency based on workload
  Trigger: continuously
  Response time: microseconds
  Performance impact: proportional to frequency reduction

Level 1 (Warning): Reduce to lower DVFS OPP
  Trigger: Tj > 85C (example threshold)
  Action: Cap maximum frequency to nominal OPP
  Performance impact: ~20-30% reduction

Level 2 (Critical): Aggressive throttling
  Trigger: Tj > 95C
  Action: Force lowest DVFS OPP, disable Turbo
  Performance impact: ~50-70% reduction

Level 3 (Emergency): Shutdown
  Trigger: Tj > 105C
  Action: Immediately shut down chip (hardware-triggered, not software)
  Purpose: prevent permanent damage
  
Recovery: resume when temperature drops below hysteresis threshold
  (e.g., Level 2 activates at 95C, deactivates at 90C -- 5C hysteresis
   prevents oscillation)
```

### 6.4 Hotspot Analysis

```
Not all parts of the chip have equal power density:
  - CPU ALU: 2-5 W/mm^2 (very hot)
  - SRAM arrays: 0.5-1 W/mm^2 (moderate)
  - IO pads: 0.1-0.3 W/mm^2 (cool)
  
Hotspot mitigation in floorplanning:
  1. Don't place high-power blocks adjacent to each other
  2. Interleave hot blocks with cool blocks (SRAM near ALU)
  3. Place critical blocks near heat extraction path (center for flip-chip,
     edges for wire-bond)
  4. Use thermal vias (through-silicon vias not for signals, just for heat)
  
Thermal simulation tools: ANSYS Icepak, Cadence Celsius, Synopsys IC Compiler thermal
```

---

## 7. Advanced Power Analysis Topics

### 7.1 Power Grid Noise and Its Impact on Timing

IR drop reduces the effective Vdd seen by cells, which slows them down:

```
Gate delay sensitivity to Vdd:
  dT/dVdd ~ -T / (Vdd - Vth)  (negative because lower Vdd -> slower)

For Vdd = 0.9V, Vth = 0.3V:
  Relative delay change per mV of IR drop = 1 / (900 - 300) = 0.167%/mV

For 50 mV IR drop:
  Delay increase = 50 * 0.167% = 8.3% slowdown

On a critical path with 500 ps delay:
  Extra delay = 0.083 * 500 = 41.5 ps
  This can easily cause a timing violation!
```

**How tools handle this:**
```
Static timing analysis (STA) with IR drop derating:
  1. Run IR drop analysis (Voltus/RedHawk) -> per-instance voltage map
  2. Feed voltage map back to STA (PrimeTime) -> per-instance derating
  3. STA uses the actual Vdd each cell sees to compute delay
  
  This is called "voltage-aware STA" or "IR-aware timing"
  
Without IR-aware timing: you must add extra margin (pessimistic) to timing constraints
With IR-aware timing: margin is reduced because you know the actual voltage per cell
```

### 7.2 Power vs Performance Trade-off Visualization

```
The Power-Performance design space:

  Power (W)
     |
  5W |  o Turbo (1.05V, 2.8 GHz)
     |
  4W |
     |
  3W |     o Nominal (0.90V, 2.0 GHz)
     |
  2W |
     |         o SVS (0.75V, 1.2 GHz)
  1W |
     |              o Low Power (0.55V, 0.4 GHz)
     |
     +-------|-------|-------|-------|-------> Performance (GHz)
            0.5     1.0     1.5     2.0

  The curve follows: P ~ V^2 * f ~ V^2 * (V-Vth)^alpha / V
  
  Energy efficiency (GOPS/W) is maximized at the LOWEST voltage point
  that still meets the throughput requirement.
```

### 7.3 Chip-Package-System Power Integrity

```
Complete power delivery network (PDN):

  VRM (Voltage Regulator Module)
    |
  [PCB trace + plane: R_pcb, L_pcb, C_pcb]
    |
  Package via/bump
    |
  [Package trace/plane: R_pkg, L_pkg, C_pkg]
    |
  C4 bump / micro-bump
    |
  [On-die PDN: R_die, L_die, C_die]
    |
  Standard cell

PDN impedance target (Zmax):
  Z_target = delta_V_max / I_max

  For delta_V = 50 mV, I_max = 5A:
  Z_target = 50e-3 / 5 = 10 mOhm

  This impedance target must be met from DC to ~10 GHz.
  Each level of the PDN hierarchy covers a frequency band.
```

---

## 8. Interview Questions and Answers

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

## 9. Summary: Power Analysis Checklist for Tapeout

```
Pre-Signoff Checklist:

[ ] Average power within budget (per domain and total)
    - Corner: TT, nominal voltage, 85C
    - Activity: SAIF from representative workload, >80% annotation coverage

[ ] Peak power within package limit
    - Corner: TT (or FF for worst-case), nominal voltage
    - Activity: VCD from worst-case scenario
    
[ ] Static IR drop < 5% Vdd everywhere
    - Fix: widen stripes, add vias, improve grid density

[ ] Dynamic IR drop < 10% Vdd everywhere
    - Fix: add decap, stagger clock edges, improve grid

[ ] Electromigration: all wires pass for 10-year lifetime at 105C
    - Fix: widen violating wires, add parallel paths

[ ] Voltage-aware STA: timing clean with IR drop derating
    - Fix: upsize cells, swap Vt, add stripes in critical regions

[ ] Leakage at worst-case temperature: no thermal runaway risk
    - Corner: FF, max voltage, 125C
    - Fix: Vt optimization, power gating

[ ] Power domain integrity: all domains have correct isolation,
    retention, level shifting (from UPF verification)

[ ] Multi-mode analysis: check power for all operating modes
    (active, idle, sleep, deep sleep, retention)
```

---

## Numbers to Memorize -- Power Analysis Quick Reference

| Quantity | Value | Why it matters |
|----------|-------|----------------|
| Dynamic power equation | P = alpha x C x V^2 x f | Quadratic voltage dependence is the key lever for power reduction |
| Leakage power equation | P = I_leak x VDD | Dominates at idle; doubles roughly every 10C temperature increase |
| Dynamic/leakage split at N5 | 70/30 (high perf), 40/60 (low power) | Determines which optimization strategy matters most |
| IR drop budget (static) | <=5% VDD | Average voltage drop; exceeded means timing failures |
| IR drop budget (dynamic) | <=10% VDD | Peak transient droop at clock edges; requires decap to fix |
| EM lifetime target | 10 years at 105C junction | Industry-standard reliability requirement for wire integrity |
| Black's equation MTTF | A x J^(-n) x exp(Ea/kT), n=1-2, Ea=0.5-0.7 eV | Higher current density (J) and temperature (T) exponentially reduce lifetime |
| Typical SoC TDP (mobile) | 2-15 W | Constrained by battery and passive cooling; sets total power budget |
| Typical SoC TDP (desktop) | 65-250 W | Active cooling budget; determines heatsink/fan design |
| Typical SoC TDP (AI accelerator) | 300-1000 W | Requires liquid or advanced cooling; drives rack-level power design |
| Power density limit (air cooling) | ~100 W/cm^2 | Exceeding this causes thermal throttling or reliability issues |
| Power density limit (liquid cooling) | ~300 W/cm^2 | Enables high-performance chips but adds cost and complexity |
| Clock power fraction | 30-50% of total dynamic power | Clock is the single largest power consumer; clock gating is essential |
| Memory (SRAM) power fraction | 15-25% of total | Large arrays contribute significant dynamic and leakage power |
| Decap cell insertion | 5-15% of total cell count | Provides local charge for transient current; trades area for IR drop margin |
| PrimeTime PX accuracy vs silicon | +/-10-15% | Gate-level power estimation; sufficient for budgeting but not final signoff |
| Voltus/RedHawk accuracy vs silicon | +/-5-10% | Post-layout power and IR drop; used for final signoff |
| Vectorless activity estimation | +/-20-30% vs VCD | Useful for early estimation; too inaccurate for signoff |
| TCF (Toggle Count Format) average activity factor | 5-20% | Typical switching activity; clock nets are 100%, random data ~50% |
| Typical SAIF metrics | Toggle rate, static probability per net | Captured during simulation; annotation coverage >80% needed for signoff |
| Power gating inrush current | 2-5x steady-state leakage | Turning on a power-gated domain causes a current surge; must size sleep transistors for peak |
| DVFS range | 0.6V-1.2V, 200 MHz - 3 GHz (typical) | Dynamic scaling saves power by matching voltage to required performance |
| Voltage scaling power benefit | Halving V -> 4x power reduction (quadratic) | Most effective power lever; why near-threshold computing is attractive |
| Frequency scaling power benefit | Halving f -> 2x power reduction (linear) | Linear benefit; less effective than voltage scaling but easier to implement |

---

*This document targets senior-engineer / staff-level ASIC power interview preparation.
Cross-reference with Power_Fundamentals.md, Power_Reduction_Techniques.md, and
UPF_Power_Intent.md.*
