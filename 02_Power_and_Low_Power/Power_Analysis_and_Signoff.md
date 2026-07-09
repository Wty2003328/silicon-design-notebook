# Power Analysis and Signoff for Digital IC / ASIC Design

## 1. Power Analysis Methodology Overview

Power analysis occurs at every stage of the design flow. Each stage trades speed for accuracy:

```ascii-graph
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
```verilog
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
```text
For net alu_result[0]:
  Static Probability (SP) = T1 / DURATION               = 1150000 / 2000000 = 0.575
  Toggle Rate (TR)        = TC / DURATION               = 2800    / 2000000 = 0.0014 transitions/ps
  Toggle Count per cycle  = TC / num_clk_cycles         = 2800    / 2000    = 1.4 transitions/cycle
  Switching Activity (SA) = TC / (2 * num_clk_cycles)   = 2800    / 4000    = 0.7
  
  (SA > 0.5 means more than one toggle per clock period on average -- possible
   if there are glitches within a cycle)
```

### 2.2 Forward vs Backward SAIF Annotation

**Forward annotation:**
```text
1. Simulate RTL
2. Dump SAIF at RTL net names
3. During power analysis on gate-level netlist:
   - Tool maps RTL net names to gate-level net names
   - Mapping is imperfect (synthesis may rename/restructure)
   - Unmapped nets use default activity -> inaccuracy
   
Advantage:    RTL simulation is fast (10-100x faster than gate-level)
Disadvantage: Naming mismatch, missing glitch activity
```

**Backward annotation:**
1. Run power tool on gate-level netlist
2. Tool generates a SAIF template with ALL gate-level net names
3. Simulate gate-level netlist, dump SAIF matching the template
4. Re-run power tool with filled-in SAIF

- **Advantage** — perfect name matching, captures glitch activity
- **Disadvantage** — gate-level simulation is slow, SAIF file is large

**Practical recommendation:**
- Use forward annotation for early estimation (RTL development phase)
- Use backward annotation for signoff (gate-level SAIF for final power numbers)
- For very large designs: hybrid (gate-level SAIF on critical blocks, vectorless for rest)

### 2.3 VCD (Value Change Dump)

VCD records EVERY signal transition with a timestamp. This enables time-based power analysis.

```verilog
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
For a design with 10M nets, toggling at average 0.1 transitions/ns, over 10us:
Total transitions = 10M * 0.1 * 10,000 = 10 billion transitions
Each transition: ~20 bytes (timestamp + signal ID + value)
VCD size = 10e9 * 20 = 200 GB

**This is impractical. Solutions:**
   1. FSDB (Synopsys): compressed binary format, 5-10x smaller
2. Selective dumping: only dump signals in the block under analysis
3. Shorter simulation window (but must be representative)
4. SAIF instead of VCD (for average power, not time-based)

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
```text
1. Assign default toggle rate (TR) and static probability (SP) to primary inputs
   Typical defaults:
     SP = 0.5
     TR = 0.1 (one transition every 10 clock cycles)

2. Propagate through the netlist:
   For an AND gate with inputs A (SP_A, TR_A) and B (SP_B, TR_B):
     SP_out = SP_A * SP_B
     TR_out = (SP_A * TR_B) + (SP_B * TR_A)  (approximate, assumes independence)
   
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

```text
  Package Pin (Vdd = 0.900V)
      |
      |  [R_package]      ~5-20 mOhm
      v
  Chip Pad (0.895V)
      |
      |  [R_pad + R_bump] ~10-50 mOhm
      v
  Top Metal Ring (0.890V)
      |
      |  [R_stripe_M9]    ~0.1-1 ohm/mm
      v
    [R_via_array]
      |
      |  [R_stripe_M7]
      v
    [R_via_array]
      |
      |  [R_stripe_M5]
      v
    ... more metal layers ...
      |
  Standard Cell Rail (0.855V)
      |
      |  [R_local]        ~1-10 ohm
      v
  Transistor Source (0.850V)

  Total IR drop: 0.900V - 0.850V = 50 mV (5.6% of Vdd)
```

### 3.2 Static IR Drop

Static IR drop assumes DC (steady-state) current. It is a pure resistive network analysis:

**The resistor mesh model:**
```ascii-graph
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
```text
Static IR drop target: < 5% of Vdd (some companies: < 3%)

For Vdd = 0.9V:
  - 5% maximum IR drop = 45 mV
  - 3% maximum IR drop = 27 mV
```

### 3.3 Dynamic IR Drop

Dynamic IR drop is MUCH worse than static because of transient current demands:

**Why dynamic IR drop peaks at clock edges:**
```text
At a clock edge, ALL flip-flops in the design sample simultaneously.
This creates a massive, near-simultaneous current demand:

  I_peak = N_switching_FFs * C_FF * Vdd * f_local

For a region with 10K FFs and 30% switching activity:
  I_peak = 3000 * 20 fF * 0.9 V / (0.5 ns half-period)
         = 3000 * 20e-15 * 0.9 / 0.5e-9
         = 3000 * 36e-6
         = 108 mA locally!

This transient current passes through grid resistance AND inductance:
  V_drop = (I * R) + (L * dI/dt)

Note: The L*dI/dt component can be 2-3x worse than the I*R component.
```

**Dynamic IR drop characteristics:**
```ascii-graph
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
Dynamic IR drop < 10% of Vdd (typical)
Some high-performance designs: < 8%
- **For Vdd** = `0.9V: maximum dynamic drop = 90 mV (10%)`

**If dynamic IR drop exceeds spec:**
   - Timing failures (cells see lower Vdd -> slower -> setup violations)
   - Functional failures at extreme drop (below noise margin)
   - Reliability issues (oxide stress, hot carrier injection from recovery overshoot)

### 3.4 Decoupling Capacitors (Decap)

Decaps provide local charge storage to supply transient current demand, reducing
dynamic IR drop:

```text
Frequency domain analysis of decap effectiveness:

1. On-die MOS-cap (gate capacitance of NMOS):
   - Effective up to: ~1-5 GHz
   - Capacitance:     ~1-5 nF/mm^2 in modern processes
   - Placement:       Whitespace (filler cells with decap)
    
2. Explicit decap cells in standard cell library:
   - Capacitance:     2-10 fF per cell (varies by size)
   - Placement:       Where IR drop is worst
   - Area cost:       Typically 5-10% of chip area dedicated to decap
    
3. Package decap:
   - Type:            Surface mount capacitors on package substrate
   - Capacitance:     10-100 nF typical
   - Effective up to: ~100-500 MHz (limited by package inductance)
    
4. PCB decap:
   - Type:            Bulk capacitors on the board
   - Capacitance:     1-100 uF
   - Effective up to: ~10-50 MHz only
    
Each level covers a different frequency band:
  
Frequency:  1Hz    1kHz   1MHz   100MHz  1GHz   10GHz
            |------|------|------|-------|------|
            [PCB decap                  ]
                   [Package decap       ]
                          [On-die decap         ]
                                 [Intrinsic cell cap]
```

**Decap sizing calculation:**
**Given:**
   - Transient current demand: I_peak = 200 mA for 200 ps duration
   - Allowable voltage droop: delta_V = 50 mV

**Required local decap:**
   - C = I * dt / dV = 200e-3 * 200e-12 / 50e-3 = 800 pF

On-die decap density: ~2 nF/mm^2
Area needed: 800pF / 2nF/mm^2 = 0.4 mm^2

If chip area is 10 mm^2, this is 4% of chip area for this region alone.

### 3.5 Electromigration (EM)

EM is the gradual displacement of metal atoms by momentum transfer from conducting
electrons. It causes wire thinning (open circuit) or hillock formation (short circuit).

**Black's Equation:**
```text
Black's Equation:
  MTTF = A * J^(-n) * exp(Ea / (k * T))

Where:
  MTTF = Mean Time To Failure
  A    = Process constant
  J    = Current density (A/cm^2)
  n    = Current density exponent (typically 1-2, usually 2 for bulk EM)
  Ea   = Activation energy (0.7-0.9 eV for Cu with liner)
  k    = Boltzmann constant (8.617e-5 eV/K)
  T    = Absolute temperature (K)
```

**EM design rules:**
DC EM limit (Jmax_DC): typically 1-3 MA/cm^2 for copper at 105C
AC EM limit (Jmax_AC): typically 2-6 MA/cm^2 (2x DC due to self-healing)

**Wire sizing for EM:**
   - Required wire width = I_rms / (Jmax * t_metal)

Example:
I_avg = 10 mA through a M5 power stripe
Jmax_DC = 2 MA/cm^2 = 2e6 A/cm^2
t_metal = 400 nm (M5 copper thickness)

Required width = 10e-3 / (2e6 * 400e-7) = 10e-3 / 0.08 = 0.125 cm = 1250 um

This means a single narrow stripe cannot carry 10 mA!
Need multiple parallel stripes or one wide stripe (~1.25 mm).

**Temperature dependence of EM:**
```text
From Black's equation, MTTF is proportional to exp(Ea / (k * T)).

For Ea = 0.7 eV:
  At T1 = 105C (378K): exp(0.7 / (8.617e-5 * 378)) = exp(21.5)
  At T2 =  85C (358K): exp(0.7 / (8.617e-5 * 358)) = exp(22.7)

MTTF ratio: 
  exp(22.7) / exp(21.5) = exp(1.2) = 3.3x

Conclusion:
  - 85C has a 3.3x longer lifetime than 105C for the same current density.
  - Every 20C increase in temperature roughly halves EM lifetime (for Ea = 0.7eV).
```

**EM at advanced nodes:**
As wire cross-sections shrink (thinner, narrower):
- Current density increases for same total current
   - Copper resistivity increases (grain boundary scattering at thin films)
   - EM becomes a tighter constraint

**Mitigation:**
   - Cobalt cap layer (better EM resistance than copper/barrier interface)
   - Ruthenium liner/barrier (thinner, more copper volume)
   - Backside power delivery (move power off the signal layers)

### 3.6 Power Grid Design

**Typical power grid structure:**
```ascii-graph
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
```text
General Rules:
  - Dedicate 5-10% of each metal layer to power routing
  - Top 2-3 metals carry most of the current (thickest, lowest resistance)
  - Alternate VDD/VSS stripes for balanced current distribution
  - Via arrays at every intersection (maximize via count for low resistance)
  - Power rings around the chip periphery (connect to bumps/pads)

Trade-offs:
  Wider stripes:
    [+] Lower IR drop (lower R)
    [+] Better EM (lower J for same I)
    [-] Less routing resource for signals
    [-] Potential routing congestion
  
  Denser stripe pitch:
    [+] More uniform voltage distribution
    [+] Better current sharing
    [-] More routing resource consumed
    [-] Diminishing returns (mesh already dense enough)

Typical Pitch Guidelines (for a medium-performance design):
  - M8/M9 (top metals): VDD+VSS stripe every 20-40 um
  - M6/M7 (mid metals): VDD+VSS stripe every 30-60 um
  - M1 (cell rail):     Continuous VDD and VSS in every cell row
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

**Example — Mobile Application Processor.** Total TDP 5 W at 85 °C junction temperature.

| Subsystem | Budget | % Total |
|---|---|---|
| CPU cluster (4× A-class cores) | 2.0 W | 40% |
| — per-core budget | 0.5 W | |
| GPU (8-core) | 1.5 W | 30% |
| Modem (5G baseband) | 0.5 W | 10% |
| Memory controller (LPDDR5) | 0.3 W | 6% |
| Display processor | 0.2 W | 4% |
| IO subsystem (USB, PCIe, UFS) | 0.2 W | 4% |
| Always-on domain (PMU, RTC, AON) | 0.05 W | 1% |
| Interconnect (NoC, bus fabric) | 0.15 W | 3% |
| Margin | 0.1 W | 2% |
| **Total** | **5.0 W** | **100%** |

### 5.2 Bottom-Up Power Verification

After design, sum actual block powers and compare with budget:

```verilog
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

Mobile SoC Power Modes:

| Mode | CPU | GPU | Modem | IO | Total | Use Case |
|---|---|---|---|---|---|---|
| Active (Max) | 2.0W | 1.5W | 0.5W | 0.2W | 5.0W | Gaming, benchmark |
| Active (Typ) | 0.8W | 0.5W | 0.3W | 0.1W | 2.0W | Normal use |
| Idle | 0.1W | Off | 0.1W | 0.05W | 0.4W | Screen on, no task |
| Light Sleep | Ret | Off | Idle | Off | 0.05W | Screen off |
| Deep Sleep | Off | Off | Idle | Off | 0.01W | Phone in pocket |
| Hibernate | Off | Off | Off | Off | 0.002W | Power button off |

Key: Ret = retention (state preserved), Off = fully power-gated

**Battery life calculation:**
```verilog
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

- **Tj** = `Ta + P * Rth_ja`

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

**Thermal resistance chain:**
   - Rth_ja = Rth_jc + Rth_cs + Rth_sa

Rth_jc = junction to case (die + package) ~ 0.5-5 C/W
Rth_cs = case to heatsink (thermal paste/pad) ~ 0.1-1 C/W
Rth_sa = heatsink to ambient (convection + radiation) ~ 1-30 C/W

**Example:**
```text
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

```verilog
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
```ascii-graph
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
- **Ring oscillator-based** — frequency decreases with temperature
- Simple, area-efficient
- Accuracy: +/- 5-10C
- Placed at expected hotspots (CPU core, GPU, memory controller)

BJT-based (bandgap reference):
- V_be decreases linearly with temperature (~-2 mV/C)
- Compare V_be with reference using ADC
- Accuracy: +/- 1-3C (better than ring osc)
- Used in most production SoCs

- **Typical placement** — 4-8 sensors per chip, at least one per major block; high-power-density designs go much denser (up to ~1 sensor per mm^2, distributed across the die)

**Throttling mechanisms (in order of severity):**
```verilog
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
  Action: Shut down non-essential domains; if temperature keeps rising,
          a last-resort hardware thermal trip (~115C) forces full chip
          shutdown independent of any software (hardware-triggered)
  Purpose: prevent permanent damage
  
Recovery: resume when temperature drops below hysteresis threshold
  (e.g., Level 2 activates at 95C, deactivates at 90C -- 5C hysteresis
   prevents oscillation)
```

### 6.4 Hotspot Analysis

Not all parts of the chip have equal power density:
- CPU ALU: 2-5 W/mm^2 (very hot)
   - SRAM arrays: 0.5-1 W/mm^2 (moderate)
   - IO pads: 0.1-0.3 W/mm^2 (cool)

**Hotspot mitigation in floorplanning:**
   1. Don't place high-power blocks adjacent to each other
2. Interleave hot blocks with cool blocks (SRAM near ALU)
3. Place critical blocks near heat extraction path (center for flip-chip,
edges for wire-bond)
4. Use thermal vias (through-silicon vias not for signals, just for heat)

Thermal simulation tools: ANSYS Icepak, Cadence Celsius, Synopsys IC Compiler thermal

---

## 7. Advanced Power Analysis Topics

### 7.1 Power Grid Noise and Its Impact on Timing

IR drop reduces the effective Vdd seen by cells, which slows them down:

**Gate delay sensitivity to Vdd:**
   - dT/dVdd ~ -T / (Vdd - Vth)  (negative because lower Vdd -> slower)

- **For Vdd** = `0.9V, Vth = 0.3V:`
   - Relative delay change per mV of IR drop = 1 / (900 - 300) = 0.167%/mV

**For 50 mV IR drop:**
   - Delay increase = 50 * 0.167% = 8.3% slowdown

**On a critical path with 500 ps delay:**
   - Extra delay = 0.083 * 500 = 41.5 ps
   - This can easily cause a timing violation!

**How tools handle this:**
```verilog
Static timing analysis (STA) with IR drop derating:
  1. Run IR drop analysis (Voltus/RedHawk) -> per-instance voltage map
  2. Feed voltage map back to STA (PrimeTime) -> per-instance derating
  3. STA uses the actual Vdd each cell sees to compute delay
  
  This is called "voltage-aware STA" or "IR-aware timing"
  
Without IR-aware timing: you must add extra margin (pessimistic) to timing constraints
With IR-aware timing: margin is reduced because you know the actual voltage per cell
```

### 7.2 Power vs Performance Trade-off Visualization

```ascii-graph
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

```verilog
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

### 7.4 Glitch Power Analysis

At <=7nm, glitch (spurious-transition) power is commonly **25-40% of dynamic power**
in datapath-heavy blocks -- large enough that ignoring it busts budgets.

```verilog
Why standard flows miss it:
  - Zero-delay RTL simulation: all input transitions arrive simultaneously
    -> NO glitches generated -> activity files underestimate datapath power
  - Vectorless propagation without timing windows: same blind spot

Analysis options (in increasing accuracy/cost):
  1. Statistical glitch estimation: power tool propagates arrival-time
     windows from STA and estimates spurious transitions per net
     (PrimePower / Joules glitch-analysis modes)
  2. SDF-annotated gate-level simulation: real glitch waveforms; slow,
     used on representative windows only
  3. Signoff tools split reported glitches into:
       transparent (full-swing)  -- full CV^2 per spurious transition
       filtered    (partial)     -- pulse narrower than gate delay; still
                                    burns partial CV^2 (not free!)

Signoff guidance: compare RTL-activity power vs SDF gate-level power on the
same window; a large gap localized in arithmetic blocks = glitch. Reduce
via path balancing, retiming, operand isolation (see
Power_Reduction_Techniques and Block_Activity_and_Power notes).
```

### 7.5 Peak Power and di/dt Signoff

Average-power signoff does not protect against transient events:

**What must be checked beyond average power:**
   1. Realistic peak window: from emulation activity of real workloads,
find the worst 1-10 us window -> drive dynamic IR analysis with it
2. Synthetic worst case: power-virus vectors (max simultaneous switching) → bounds VRM/package current, validates throttle response
3. di/dt events: domain wake-up (rush current), vector-unit turn-on,
clock-ungating of a large cluster -- each is a load STEP that excites
the package resonance (~50-300 MHz "first droop")
4. Checks: peak droop within budget WITH the droop mitigation modeled
(decap + adaptive clocking), current ramp within PMIC slew capability,
no EM overstress on the burst profile

### 7.6 Backside Power Delivery (BSPDN) -- What Changes for Power Signoff

Through 2025-2026, power delivery moved to the wafer backside on leading nodes:
**Intel 18A PowerVia** (in volume production 2025; power routed on the backside,
connecting to transistor contacts) and **TSMC A16 "Super Power Rail"** (production
2H 2026; backside network contacting source/drain directly).

```verilog
Why: frontside power grids competed with signal routing on the lower metal
layers and reached transistors through tall, resistive via stacks.

Benefits (reported):
  - Large IR-drop reduction (short, fat backside vias feed the cells)
  - Frees frontside routing tracks -> better density/congestion
    (Intel reported ~6% frequency gain attributable to PowerVia-class
    delivery plus the routing relief)

What changes for the signoff engineer:
  - PDN extraction now includes backside metal + nanoTSVs; new tech files
  - IR analysis topology changes: drop is dominated by the backside
    network and the TSV interface, not M1-M3 grid weave
  - Thermal: the silicon between devices and the backside grid is thinned;
    heat paths and thermal-sensor placement assumptions change -- thermal
    and IR signoff become more coupled
  - Decap strategy shifts (backside capacitance options, different
    effective inductance to the bumps)
```

---

## 8. Summary: Power Analysis Checklist for Tapeout

```verilog
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
