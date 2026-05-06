# Logic Synthesis and Optimization -- Senior Engineer Deep Dive

> Target audience: Engineers preparing for senior-level IC design interviews at
> Apple, NVIDIA, AMD, Intel, Qualcomm, Broadcom, MediaTek, etc.

---

## Table of Contents

1. [RTL-to-Gates Flow](#1-rtl-to-gates-flow)
2. [SDC Constraints Deep Dive](#2-sdc-constraints-deep-dive)
3. [Optimization Techniques](#3-optimization-techniques)
4. [Technology Mapping](#4-technology-mapping)
5. [Timing Closure Methodology](#5-timing-closure-methodology)
6. [Area Optimization](#6-area-optimization)
7. [Power Optimization in Synthesis](#7-power-optimization-in-synthesis)
8. [Design Compiler vs Genus Comparison](#8-design-compiler-vs-genus-comparison)
9. [Interview Q&A](#9-interview-qa)

---

## 1. RTL-to-Gates Flow

### 1.1 Three Phases of Synthesis

```
  RTL-to-Gates Pipeline
  =====================

  ┌─────────────────┐
  │   RTL (Verilog/  │
  │   SystemVerilog/ │
  │   VHDL)          │
  └────────┬─────────┘
           │
           v
  ┌─────────────────────────────────────────────────────┐
  │  PHASE 1: TRANSLATION (RTL → GTECH/Generic)         │
  │                                                     │
  │  - Parse HDL into internal representation           │
  │  - Infer sequential elements (FFs, latches, RAMs)   │
  │  - Build technology-independent Boolean network      │
  │  - GTECH cells: generic AND, OR, MUX, FF, etc.     │
  │  - No timing/area info yet                          │
  └────────────────────┬────────────────────────────────┘
                       │
                       v
  ┌─────────────────────────────────────────────────────┐
  │  PHASE 2: OPTIMIZATION (Boolean + Structural)        │
  │                                                     │
  │  - Two-level optimization (espresso-like minimize)  │
  │  - Multi-level optimization (algebraic/Boolean)     │
  │  - Factoring, decomposition, substitution           │
  │  - Retiming (moving FFs across combinational logic) │
  │  - Resource sharing (ALU reuse across MUX branches) │
  │  - Datapath optimization (carry-save, wallace tree) │
  │  - Still technology-independent                     │
  └────────────────────┬────────────────────────────────┘
                       │
                       v
  ┌─────────────────────────────────────────────────────┐
  │  PHASE 3: MAPPING (Generic → Technology-Specific)    │
  │                                                     │
  │  - Map Boolean network to target library cells      │
  │  - DAG covering / tree covering algorithms          │
  │  - Boolean matching (NPN equivalence)               │
  │  - Cell selection (drive strength, Vt flavor)       │
  │  - Timing-driven: pick faster cells on critical path│
  │  - Area-driven: pick smallest cells off critical    │
  │  - Design Rule fixing (max transition, cap, fanout) │
  └────────────────────┬────────────────────────────────┘
                       │
                       v
  ┌─────────────────┐
  │  Gate-Level      │
  │  Netlist (.v)    │
  └─────────────────┘
```

### 1.2 Synopsys Design Compiler Flow

```tcl
# ===== Synopsys Design Compiler (DC) Flow =====

# 1. Setup
set_app_var target_library "tsmc7ff_sc_rvt_ss_0p72v_m40c.db"
set_app_var link_library   "* tsmc7ff_sc_rvt_ss_0p72v_m40c.db"
set_app_var search_path    "./libs ./rtl"

# 2. Read RTL
read_verilog {top.v sub_module_a.v sub_module_b.v}
# Or: analyze + elaborate (preferred for parameterized designs)
analyze -format sverilog {top.sv sub_a.sv sub_b.sv}
elaborate top -parameters {DATA_WIDTH=64, DEPTH=16}

# 3. Link design
current_design top
link

# 4. Apply constraints (SDC)
source ./constraints/top.sdc

# 5. Compile
# Basic: compile -map_effort high
# Advanced: compile_ultra (includes datapath opt, boolean opt, auto-ungroup)
compile_ultra

# 6. Incremental optimization (optional)
compile_ultra -incremental

# 7. Reports
report_timing -nworst 10 -max_paths 50 > reports/timing.rpt
report_area                              > reports/area.rpt
report_power                             > reports/power.rpt
report_constraint -all_violators         > reports/constraints.rpt
report_qor                              > reports/qor.rpt

# 8. Write outputs
write -format verilog -hierarchy -output netlist/top.v
write_sdc -version 2.1 output/top.sdc
write -format ddc -hierarchy -output ddc/top.ddc
```

### 1.3 Cadence Genus Flow

```tcl
# ===== Cadence Genus Synthesis Flow =====

# 1. Setup
set_db init_lib_search_path ./libs
set_db library {tsmc7ff_sc_rvt_ss_0p72v_m40c.lib}

# 2. Read RTL
read_hdl -sv {top.sv sub_a.sv sub_b.sv}

# 3. Elaborate
elaborate top

# 4. Apply constraints
read_sdc ./constraints/top.sdc
# Or individual constraint commands

# 5. Three-step synthesis
syn_generic          ;# Technology-independent optimization
                      # (Boolean optimization, resource sharing)

syn_map              ;# Technology mapping to library cells
                      # (cell selection, drive strength)

syn_opt              ;# Post-mapping optimization
                      # (timing closure, DRV fixing)

# Optional: incremental optimization
syn_opt -incr

# 6. Reports
report_timing -nworst 10 > reports/timing.rpt
report_area              > reports/area.rpt
report_power             > reports/power.rpt
report_qor               > reports/qor.rpt

# 7. Write outputs
write_hdl > netlist/top.v
write_sdc > output/top.sdc
write_design -innovus -basename output/top
```

### 1.4 What Happens During Each Phase

**Phase 1 -- Translation:**

```
  RTL code:
    always @(posedge clk)
      if (sel)
        q <= a + b;
      else
        q <= c & d;

  GTECH representation:
    GTECH_ADD(a, b)  → sum
    GTECH_AND(c, d)  → and_out
    GTECH_MUX(sel, and_out, sum) → mux_out
    GTECH_FD(clk, mux_out) → q
```

**Phase 2 -- Optimization (selected techniques):**

```
  Boolean optimization example:
    Original: f = abc + abd + ab'c + ab'd
    Factor:   f = a(bc + bd + b'c + b'd)
            = a((b+b')(c+d))       -- wait, let's do this properly
            = a(bc + bd + b'c + b'd)
            = a(c(b+b') + d(b+b'))
            = a(c + d)
    Simplified: 1 AND + 1 OR vs original 4 ANDs + 1 OR (3 inputs each)

  Structuring:
    Decompose large functions into tree of smaller gates
    for better mapping to library cells.
```

**Phase 3 -- Technology Mapping:**

```
  Generic Boolean network:
    y = (a & b) | (c & d)

  Mapped to library cells (option A -- area optimized):
    AND2_X1(a, b) → n1
    AND2_X1(c, d) → n2
    OR2_X1(n1, n2) → y
    Area: 3 cells, ~6 gate equivalents

  Mapped to library cells (option B -- timing optimized):
    AO22_X2(a, b, c, d) → y    [AND-OR complex gate]
    Area: 1 cell, ~5 gate equivalents, fewer stages = faster
```

---

## 2. SDC Constraints Deep Dive

SDC (Synopsys Design Constraints) is the industry standard for specifying
timing constraints. Every command below is critical for senior-level understanding.

### 2.1 create_clock

```tcl
# Basic clock: 500 MHz, 50% duty cycle
create_clock -name sys_clk -period 2.0 [get_ports clk]
#             ^name         ^2ns=500MHz  ^clock source pin

# Clock with non-50% duty cycle (60% high, 40% low)
create_clock -name asym_clk -period 10.0 -waveform {0 6.0} [get_ports clk2]
#                                         ^rise at 0, fall at 6ns
#                                          → 60% duty cycle

# Virtual clock (no physical source pin -- for IO delay constraints)
create_clock -name vclk_ext -period 5.0
# No [get_ports ...] → virtual clock
# Used as reference for set_input_delay / set_output_delay
# when the external interface clock is not a port of this block
```

**When to use virtual clocks:**
- Constraining IO timing when the external clock is not a port of the block
- Specifying interface timing to an external chip or FPGA
- Creating a reference for IO timing that differs from the internal clock

### 2.2 create_generated_clock

```tcl
# Divide-by-2 clock from a flip-flop
create_generated_clock -name clk_div2 \
    -source [get_ports clk] \
    -divide_by 2 \
    [get_pins div_ff/Q]

# Multiply-by-3 (PLL output)
create_generated_clock -name pll_clk_3x \
    -source [get_pins pll/clk_in] \
    -multiply_by 3 \
    [get_pins pll/clk_out]

# Edge-based definition (for complex waveforms)
# Source clock: period=10, edges at 0,5,10,15,20...
# Generated clock uses source edges 1,3,5 (= times 0,10,20)
# → period = 20ns (divide by 2)
create_generated_clock -name clk_div2_edges \
    -source [get_ports clk] \
    -edges {1 3 5} \
    [get_pins mux/Y]

# Edge with shift (for phase-shifted clocks)
create_generated_clock -name clk_shifted \
    -source [get_ports clk] \
    -edges {1 2 3} \
    -edge_shift {0 0 0} \
    [get_pins buf/Y]
# edge_shift adds delay to each edge: {rise_shift fall_shift next_rise_shift}
```

**Why generated clocks matter:** The synthesis and STA tools must know the
exact phase relationship between clocks. Generated clocks maintain a defined
relationship to their source, enabling proper inter-clock timing analysis.
If you define a divided clock as `create_clock` instead of
`create_generated_clock`, the tool treats it as asynchronous to the source --
wrong!

### 2.3 set_clock_uncertainty

```tcl
# Setup uncertainty (pessimistic -- adds to required time)
set_clock_uncertainty -setup 0.15 [get_clocks sys_clk]
# Tsetup_slack = T_required - T_arrival
# T_required = T_period - T_setup - T_uncertainty
# → uncertainty REDUCES available timing margin

# Hold uncertainty
set_clock_uncertainty -hold 0.05 [get_clocks sys_clk]
# Adds margin to hold check

# Inter-clock uncertainty (between two different clocks)
set_clock_uncertainty -setup 0.25 \
    -from [get_clocks clk_a] \
    -to [get_clocks clk_b]
# Accounts for PLL jitter, clock tree skew mismatch, etc.

# Pre-CTS vs Post-CTS:
# Pre-CTS:  uncertainty = jitter + estimated_skew (larger, ~200-300ps)
# Post-CTS: uncertainty = jitter only (skew is actual, ~50-100ps)
```

**Typical values:**

```
  Component              Value (7nm example)
  ─────────────────────  ──────────────────
  PLL jitter             20-50 ps
  Pre-CTS clock skew     100-200 ps (estimated)
  Post-CTS residual skew 10-30 ps
  OCV derating           Applied separately (not in uncertainty)
  
  Pre-CTS setup uncertainty ≈ 150-250 ps
  Post-CTS setup uncertainty ≈ 50-80 ps
```

### 2.4 set_clock_latency

```tcl
# Source latency: delay from actual clock source to clock definition point
# (e.g., delay through PLL, off-chip oscillator to chip pin)
set_clock_latency -source -max 1.5 [get_clocks sys_clk]
set_clock_latency -source -min 1.2 [get_clocks sys_clk]

# Network latency: delay from clock definition point to flip-flop clock pins
# (estimated pre-CTS, replaced by actual propagated latency post-CTS)
set_clock_latency -max 0.8 [get_clocks sys_clk]    ;# network latency (default)
set_clock_latency -min 0.6 [get_clocks sys_clk]

# Post-CTS: set_propagated_clock replaces network latency with actual delays
set_propagated_clock [get_clocks sys_clk]
```

### 2.5 set_input_delay / set_output_delay

```tcl
# ──────────────────────────────────────────────────────
# set_input_delay: time from clock edge to data arriving at input port
# ──────────────────────────────────────────────────────

# Standard input delay
set_input_delay -clock sys_clk -max 1.2 [get_ports data_in]
set_input_delay -clock sys_clk -min 0.3 [get_ports data_in]
# -max used for setup analysis, -min for hold analysis

# DDR interface: data valid on both clock edges
set_input_delay -clock ddr_clk -max 0.8 [get_ports ddr_data]
set_input_delay -clock ddr_clk -max 0.8 -clock_fall -add_delay [get_ports ddr_data]
# -clock_fall: referenced to falling edge
# -add_delay: ADD this constraint (don't replace the rising edge one)

# ──────────────────────────────────────────────────────
# set_output_delay: time required at output port BEFORE next clock edge
# ──────────────────────────────────────────────────────

set_output_delay -clock sys_clk -max 1.0 [get_ports data_out]
set_output_delay -clock sys_clk -min 0.2 [get_ports data_out]
# -max: setup requirement at receiving end
# -min: hold requirement at receiving end
```

**Understanding the timing budget:**

```
  For input path:
  ═══════════════════════════════════════════════════
  
  External          │        This Block
  ──────────────────┼──────────────────────────────
                    │
  [ext_FF] ──delay──┼──> [input port] ──> [comb] ──> [int_FF]
                    │
  ←── input_delay ──→←── available for internal logic ──→
                    
  T_internal_max = T_period - input_delay_max - setup_time - uncertainty

  Example: period=2ns, input_delay_max=1.2ns, setup=0.05ns, uncertainty=0.1ns
  T_internal_max = 2.0 - 1.2 - 0.05 - 0.1 = 0.65 ns for internal logic


  For output path:
  ═══════════════════════════════════════════════════
  
  This Block                │        External
  ──────────────────────────┼──────────────────────
                            │
  [int_FF] ──> [comb] ──> [output port] ──delay──> [ext_FF]
                            │
  ←── available ──→←── output_delay ──→
  
  T_internal_max = T_period - output_delay_max - setup_time - uncertainty
```

### 2.6 set_false_path

```tcl
# Between asynchronous clock domains (CDC paths)
set_false_path -from [get_clocks clk_a] -to [get_clocks clk_b]
set_false_path -from [get_clocks clk_b] -to [get_clocks clk_a]

# Test mode paths (not functional timing-critical)
set_false_path -from [get_ports scan_enable]
set_false_path -from [get_ports test_mode]

# MUX-selected paths (only one active at a time)
set_false_path -from [get_cells mux_sel_reg] -to [get_cells output_reg]
# (if mux_sel is static configuration)

# Through specific pins
set_false_path -through [get_pins mux/S]
# Use -through carefully -- it can mask real paths

# Reset paths (async reset is not timing-critical for setup)
set_false_path -from [get_ports rst_n]
```

**When NOT to use false path:**
- Do NOT false-path CDC paths that need max_delay constraints for
  reconvergence or MTBF. Use set_max_delay -datapath_only instead.
- Do NOT false-path paths just because they have large slack -- they
  might become critical after optimization.

### 2.7 set_multicycle_path

This is the **most commonly misunderstood** SDC constraint.

```tcl
# Multicycle path: data takes N clock cycles to propagate
# DEFAULT: -setup 1 (single cycle), -hold 0

# Setup multicycle of 2: data is captured 2 cycles after launch
set_multicycle_path 2 -setup -from [get_cells reg_a] -to [get_cells reg_b]

# CRITICAL: Hold adjustment is almost always needed!
# Without hold adjustment, hold check moves to cycle (N-1) = 1
# This is usually wrong -- hold should be checked at the launch edge
set_multicycle_path 1 -hold -from [get_cells reg_a] -to [get_cells reg_b]
```

**Detailed timing diagram for multicycle = 2:**

```
  Clock period = T

  Launch edge    Capture edge (default)    Capture edge (MCP setup 2)
       │                  │                         │
       v                  v                         v
  ───┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐    ┌────┐
     └────┘    └────┘    └────┘    └────┘    └────┘    └──
       0        T        2T        3T        4T       
       ^                  ^                   ^
    Launch             Default              MCP=2
    edge              capture             capture
                      (1 cycle)           (2 cycles)

  Setup check with MCP=2:
    Data must arrive by 2T - Tsetup (instead of T - Tsetup)
    Available time = 2T (twice as much slack!)

  Hold check WITHOUT -hold adjustment:
    Hold check moves to: capture_edge - (N-1)*T = 2T - 1*T = T
    This means hold is checked at edge T, not edge 0!
    → The tool requires data to be stable past edge T
    → This is overly pessimistic and often creates violations

  Hold check WITH "set_multicycle_path 1 -hold":
    Hold check: capture_edge - N_hold*T = 2T - 1*T = T
    Wait, the -hold N means subtract N from the capture edge for hold:
    
    Correct formula:
      Setup capture edge = launch_edge + N_setup * T
      Hold check edge = setup_capture_edge - N_hold * T
    
    Default: N_setup=1, N_hold=0 → hold at launch_edge + T - 0 = launch+T
    NO! Let me be precise:

    For MCP setup=2, hold=1:
      Setup check: launch + 2T (data must arrive before this)
      Hold check:  launch + 2T - 1T = launch + T
      
    Hmm, still at T. For most multicycle paths, we want hold at launch edge:
      set_multicycle_path 2 -setup → capture at launch + 2T
      set_multicycle_path 1 -hold  → hold at launch + 2T - 1*T = launch + T
    
    Still not at launch edge. Need:
      set_multicycle_path 2 -hold → hold at launch + 2T - 2T = launch
      NO -- -hold value cannot exceed -setup value minus 1... actually:

  CORRECT RULE:
  ─────────────────────────────────────────────────────────
  Setup MCP = N means: capture edge = launch + N * T
  Hold MCP = M means:  hold check edge = (launch + N*T) - M*T = launch + (N-M)*T

  To check hold at the launch edge: M = N → but convention is M = N-1:
    set_multicycle_path N -setup
    set_multicycle_path (N-1) -hold
    → hold at launch + (N - (N-1))*T = launch + T

  That's checking hold one cycle after launch -- the standard convention.
  To check at launch edge exactly, use M = N:
    set_multicycle_path N -hold (same as -setup value)
    → hold at launch + (N-N)*T = launch edge

  INDUSTRY STANDARD PRACTICE:
    set_multicycle_path N -setup -from A -to B
    set_multicycle_path (N-1) -hold -from A -to B
    
  This gives:
    Setup: N cycles of timing budget
    Hold: checked 1 cycle after launch (safe for most designs)
  ─────────────────────────────────────────────────────────
```

**Worked example:**

```
  Clock period = 2 ns (500 MHz)
  MCP = 3 (data valid every 3rd cycle)

  set_multicycle_path 3 -setup -from [get_cells slow_reg] -to [get_cells fast_reg]
  set_multicycle_path 2 -hold  -from [get_cells slow_reg] -to [get_cells fast_reg]

  Setup budget: 3 x 2ns = 6ns (minus setup time and uncertainty)
  Hold check: at launch + (3-2)*2ns = launch + 2ns

  If combinational delay from slow_reg to fast_reg = 4.5 ns:
    Setup slack = 6.0 - 0.05 - 0.1 - 4.5 = 1.35 ns ✓ (positive = good)
    Without MCP: slack = 2.0 - 0.05 - 0.1 - 4.5 = -2.65 ns ✗ (violation!)
```

### 2.8 set_max_delay / set_min_delay

```tcl
# For CDC paths where you need bounded delay (not false path)
set_max_delay 3.0 -datapath_only \
    -from [get_cells cdc_src_reg] \
    -to [get_cells cdc_dst_reg]

# -datapath_only: ignores clock path delays in the calculation
# This is CRITICAL for CDC -- you want to constrain only the
# data path delay, not include clock skew effects

# Without -datapath_only:
#   max_delay check includes: clock_path_to_launch + data_delay - clock_path_to_capture
# With -datapath_only:
#   max_delay check: data_delay only (what we want for CDC)

set_min_delay 0.5 \
    -from [get_cells fast_reg] \
    -to [get_cells slow_reg]
# Ensures minimum propagation delay (used for pulse-width guarantees, etc.)
```

### 2.9 set_clock_groups

```tcl
# ──────────────────────────────────────────────────────
# ASYNCHRONOUS: clocks exist simultaneously, no phase relationship
# ──────────────────────────────────────────────────────
set_clock_groups -asynchronous \
    -group [get_clocks {sys_clk pll_clk_div2}] \
    -group [get_clocks {usb_clk}] \
    -group [get_clocks {pcie_clk}]
# No timing paths analyzed between groups
# Both clock trees ARE built (physical coexistence)
# Used for: truly independent clock sources

# ──────────────────────────────────────────────────────
# PHYSICALLY EXCLUSIVE: clocks CANNOT coexist on silicon
# ──────────────────────────────────────────────────────
set_clock_groups -physically_exclusive \
    -group [get_clocks func_clk] \
    -group [get_clocks test_clk]
# These are MUX-selected: only one reaches the clock tree at a time
# CTS can share resources between them
# Used for: clock MUX alternatives (test vs functional)

# ──────────────────────────────────────────────────────
# LOGICALLY EXCLUSIVE: both physically present but functionally exclusive
# ──────────────────────────────────────────────────────
set_clock_groups -logically_exclusive \
    -group [get_clocks mode_a_clk] \
    -group [get_clocks mode_b_clk]
# Both clock trees exist physically
# But design ensures only one is active via mode register
# CTS must build both trees independently
# Used for: mode-selected clocks where both propagate but only one is used
```

**Critical differences summary:**

```
  ┌──────────────────────┬──────────────┬──────────────┬──────────────┐
  │ Property             │ Asynchronous │ Phys Excl    │ Logic Excl   │
  ├──────────────────────┼──────────────┼──────────────┼──────────────┤
  │ Both on silicon?     │ YES          │ NO           │ YES          │
  ├──────────────────────┼──────────────┼──────────────┼──────────────┤
  │ Timing between?      │ NO           │ NO           │ NO           │
  ├──────────────────────┼──────────────┼──────────────┼──────────────┤
  │ Share CTS resources? │ NO           │ YES          │ NO           │
  ├──────────────────────┼──────────────┼──────────────┼──────────────┤
  │ SI/Crosstalk analysis│ YES          │ NO           │ YES          │
  │ between groups?      │              │              │              │
  ├──────────────────────┼──────────────┼──────────────┼──────────────┤
  │ Typical use case     │ Independent  │ Test vs func │ Config-mode  │
  │                      │ PLLs         │ clock mux    │ selected     │
  └──────────────────────┴──────────────┴──────────────┴──────────────┘
```

### 2.10 set_case_analysis

```tcl
# Fix a pin to a constant value for mode-specific analysis
set_case_analysis 0 [get_ports test_mode]
# All paths through test_mode=0 branch are analyzed
# Paths through test_mode=1 branch are blocked

set_case_analysis 1 [get_ports func_mode_sel]

# Useful for:
# - Analyzing specific operating modes
# - Fixing clock MUX selects during functional analysis
# - Setting configuration pins to their functional values
```

### 2.11 set_disable_timing

```tcl
# Break a timing arc through a cell
set_disable_timing [get_cells clk_mux] -from S -to Y
# Disables the select-to-output arc of the mux
# Still allows input-to-output arcs

# Use case: MUX with slow select signal that doesn't affect data timing
# Warning: can hide real timing problems -- use sparingly
```

### 2.12 Design Rule Violation (DRV) Constraints

```tcl
# Maximum transition time (slew) on any net
set_max_transition 0.2 [current_design]
# No signal should take > 200ps to transition
# Violated: long wires, high fanout, weak drivers

# Maximum capacitance on any output pin
set_max_capacitance 0.1 [current_design]
# In pF -- limits load on any driver
# Violated: many loads on one net

# Maximum fanout
set_max_fanout 20 [current_design]
# No output should drive > 20 loads
# Tool inserts buffers to fix

# These are checked as constraints during compile:
# report_constraint -all_violators shows DRV violations
```

### 2.13 Complete SDC Example: 3-Clock-Domain SoC

```tcl
# ================================================================
# SDC for a 3-clock-domain SoC with DDR and SPI interfaces
# ================================================================

# ──────── Clock Definitions ────────

# Core clock: 1 GHz
create_clock -name core_clk -period 1.0 [get_ports clk_core]

# Peripheral clock: 200 MHz (from PLL, divided from core)
create_generated_clock -name peri_clk \
    -source [get_pins pll/clk_out] \
    -divide_by 5 \
    [get_pins peri_clk_div/Q]

# DDR clock: 800 MHz (from PLL)
create_generated_clock -name ddr_clk \
    -source [get_pins pll/clk_out] \
    -multiply_by 4 -divide_by 5 \
    [get_pins ddr_pll/clk_out]

# Virtual clock for SPI interface (external, 50 MHz)
create_clock -name spi_vclk -period 20.0

# Test clock (physically exclusive with core_clk)
create_clock -name test_clk -period 10.0 [get_ports tck]

# ──────── Clock Relationships ────────

# Core and peripheral are synchronous (same PLL source)
# → no special constraint needed, tool will analyze

# DDR clock is asynchronous to core (different PLL output)
set_clock_groups -asynchronous \
    -group [get_clocks {core_clk peri_clk}] \
    -group [get_clocks ddr_clk]

# Test clock physically exclusive with functional clocks
set_clock_groups -physically_exclusive \
    -group [get_clocks test_clk] \
    -group [get_clocks {core_clk peri_clk ddr_clk}]

# ──────── Clock Uncertainty ────────

# Pre-CTS (larger margin for estimated skew)
set_clock_uncertainty -setup 0.15 [get_clocks core_clk]
set_clock_uncertainty -hold  0.05 [get_clocks core_clk]
set_clock_uncertainty -setup 0.10 [get_clocks peri_clk]
set_clock_uncertainty -hold  0.03 [get_clocks peri_clk]
set_clock_uncertainty -setup 0.12 [get_clocks ddr_clk]
set_clock_uncertainty -hold  0.04 [get_clocks ddr_clk]

# Inter-clock uncertainty (core ↔ peripheral)
set_clock_uncertainty -setup 0.20 \
    -from [get_clocks core_clk] -to [get_clocks peri_clk]
set_clock_uncertainty -setup 0.20 \
    -from [get_clocks peri_clk] -to [get_clocks core_clk]

# ──────── Clock Latency ────────

set_clock_latency -source -max 0.5 [get_clocks core_clk]
set_clock_latency -source -min 0.3 [get_clocks core_clk]
set_clock_latency 0.6 [get_clocks core_clk]   ;# estimated network latency

# ──────── IO Constraints ────────

# DDR data interface (both edges)
set_input_delay -clock ddr_clk -max 0.4 [get_ports ddr_dq[*]]
set_input_delay -clock ddr_clk -min 0.1 [get_ports ddr_dq[*]]
set_input_delay -clock ddr_clk -max 0.4 -clock_fall -add_delay [get_ports ddr_dq[*]]
set_input_delay -clock ddr_clk -min 0.1 -clock_fall -add_delay [get_ports ddr_dq[*]]

set_output_delay -clock ddr_clk -max 0.3 [get_ports ddr_dq[*]]
set_output_delay -clock ddr_clk -min 0.05 [get_ports ddr_dq[*]]
set_output_delay -clock ddr_clk -max 0.3 -clock_fall -add_delay [get_ports ddr_dq[*]]
set_output_delay -clock ddr_clk -min 0.05 -clock_fall -add_delay [get_ports ddr_dq[*]]

# SPI interface (referenced to virtual clock)
set_input_delay -clock spi_vclk -max 8.0 [get_ports spi_miso]
set_input_delay -clock spi_vclk -min 1.0 [get_ports spi_miso]
set_output_delay -clock spi_vclk -max 6.0 [get_ports {spi_mosi spi_cs_n}]
set_output_delay -clock spi_vclk -min 1.0 [get_ports {spi_mosi spi_cs_n}]

# General IO
set_input_delay -clock core_clk -max 0.5 [get_ports gpio_in[*]]
set_output_delay -clock core_clk -max 0.5 [get_ports gpio_out[*]]

# ──────── False Paths ────────

set_false_path -from [get_ports rst_n]
set_false_path -from [get_ports test_mode]

# CDC false paths (handled by synchronizers, constrained with max_delay)
# Don't false-path if you need max_delay for reconvergence!

# ──────── CDC Max Delay ────────

set_max_delay 1.5 -datapath_only \
    -from [get_clocks core_clk] -to [get_clocks ddr_clk]
set_max_delay 1.5 -datapath_only \
    -from [get_clocks ddr_clk] -to [get_clocks core_clk]

# ──────── Multicycle Paths ────────

# Slow config registers: written by peri_clk, stable for 4 core_clk cycles
set_multicycle_path 4 -setup \
    -from [get_cells config_regs/*] -to [get_cells core_logic/*]
set_multicycle_path 3 -hold \
    -from [get_cells config_regs/*] -to [get_cells core_logic/*]

# ──────── Mode Analysis ────────

set_case_analysis 0 [get_ports test_mode]
set_case_analysis 0 [get_ports scan_enable]

# ──────── Design Rules ────────

set_max_transition 0.15 [current_design]
set_max_capacitance 0.08 [current_design]
set_max_fanout 30 [current_design]

# ──────── Operating Conditions ────────

set_operating_conditions -max ss_0p72v_m40c -min ff_0p88v_125c
# Worst-case (slow) for setup, best-case (fast) for hold
```

---

## 3. Optimization Techniques

### 3.1 Ungrouping

```tcl
# Remove hierarchy boundaries to allow cross-module optimization
set_ungroup [get_designs sub_module_a]
# or during compile:
compile_ultra -ungroup_all
```

```
  Before ungrouping:
  ┌─────────────────────┐    ┌──────────────────────┐
  │  Module A            │    │  Module B             │
  │  [logic] ──> port_a ─┼───>┼─ port_b ──> [logic]  │
  │                      │    │                       │
  └─────────────────────┘    └──────────────────────┘
  Boundary ports prevent optimization across modules.

  After ungrouping:
  ┌──────────────────────────────────────────────────┐
  │  Flattened design                                 │
  │  [logic_A] ──────────────────> [logic_B]          │
  │  (no boundary, tool can optimize the connection)  │
  └──────────────────────────────────────────────────┘
```

**When to ungroup:**
- Small modules on critical paths
- Glue logic modules
- Wrapper modules with no meaningful hierarchy

**When NOT to ungroup:**
- Large modules (makes optimization intractable)
- Modules you want to preserve for ECO or debug
- Hard macros (memories, IPs)

### 3.2 Boundary Optimization

```tcl
set_boundary_optimization [get_designs sub_module] true
```

The tool can:
- Propagate constants across hierarchy boundaries
- Remove unconnected/unused ports
- Merge duplicate logic across boundaries

```
  Example: Module port tied to constant
  
  Before:
    top: assign sub_inst.config = 1'b0;
    sub: always @(*) if (config) y = a; else y = b;

  After boundary optimization:
    sub: y = b;  (config=0 propagated, dead branch removed)
```

### 3.3 Retiming

Moving registers across combinational logic to balance path delays.

```
  Forward Retiming (pipeline register moved forward):
  ═══════════════════════════════════════════════════

  Before:
    [REG] ──> [Long Comb Logic A: 3ns] ──> [Short Comb Logic B: 0.5ns] ──> [REG]
    Critical path = 3ns (Logic A limits frequency)

  After forward retiming:
    [REG] ──> [Comb Logic A part1: 1.5ns] ──> [REG] ──> [Comb Logic A part2 + B: 2ns] ──> [REG]
    Critical path = 2ns (balanced!)


  Backward Retiming (pipeline register moved backward):
  ════════════════════════════════════════════════════

  Before:
    [REG] ──> [Short Logic: 0.5ns] ──> [Long Logic: 3ns] ──> [REG]

  After backward retiming:
    [REG] ──> [Short + Long_part1: 1.75ns] ──> [REG] ──> [Long_part2: 1.75ns] ──> [REG]
```

**Constraints on retiming:**
- Cannot retime across I/O boundaries (would change interface timing)
- Cannot retime across clock domain boundaries
- Cannot retime if it changes reset behavior (initial values matter)
- Cannot retime if register has special attributes (scan, dont_touch)
- Tool must verify functional equivalence before/after

```tcl
# Enable retiming in DC
compile_ultra -retime

# In Genus
set_db design:top .retime true
syn_opt -retime
```

### 3.4 Datapath Optimization

Synthesis tools recognize arithmetic operations and implement them efficiently.

```
  Carry-Save Optimization:
  ════════════════════════

  RTL: result = a + b + c + d;

  Naive implementation (3 serial adders):
    [a+b] ──> [+c] ──> [+d] ──> result
    Delay: 3 x adder_delay (critical path through carry chains)

  Carry-save optimization:
    [CSA(a,b,c)] ──> [sum1, carry1]
    [CSA(sum1, carry1, d)] ──> [sum2, carry2]
    [CPA(sum2, carry2)] ──> result
    Delay: 2 x CSA_delay + 1 x CPA_delay
    (CSA is single-gate delay; only final CPA has carry chain)


  Wallace Tree (for multiplication):
  ═══════════════════════════════════
  
  For 8x8 multiplier:
    Naive: 8 partial products, added sequentially = 7 additions
    Wallace tree: partial products reduced in parallel stages
      Stage 1: 8 rows → 6 rows (using CSAs)
      Stage 2: 6 rows → 4 rows
      Stage 3: 4 rows → 3 rows
      Stage 4: 3 rows → 2 rows
      Final:   2 rows → 1 result (CPA)
    Depth: O(log n) vs O(n)
```

**Resource sharing:**

```verilog
  // Before resource sharing:
  always @(*) begin
      if (sel)
          result = a + b;    // Adder 1
      else
          result = c + d;    // Adder 2
  end
  // Two adders instantiated

  // After resource sharing:
  // Tool transforms to:
  //   mux_a = sel ? a : c;
  //   mux_b = sel ? b : d;
  //   result = mux_a + mux_b;    // One adder!
  // Trade-off: 2 muxes added, 1 adder removed
  // Area savings if adder >> 2 muxes (true for wide datapaths)
```

```tcl
# Enable resource sharing
set_resource_allocation shared
# or
compile_ultra  ;# does this automatically
```

### 3.5 Compile Strategies

```tcl
# ── Strategy 1: Basic compile (fast, lower QoR) ──
compile -map_effort high

# ── Strategy 2: compile_ultra (best single-pass QoR) ──
compile_ultra
# Includes: auto-ungrouping, datapath opt, boolean opt, register retiming

# ── Strategy 3: compile_ultra + incremental ──
compile_ultra
# ... fix constraints, add path groups ...
compile_ultra -incremental
# Incremental only makes local changes, won't restructure

# ── Strategy 4: Two-pass with path groups ──
compile_ultra
group_path -name critical_group \
    -from [get_cells cpu_core/*] \
    -to [get_cells cpu_core/*] \
    -weight 2.0
compile_ultra -incremental

# ── Strategy 5: Topographical mode (placement-aware) ──
compile_ultra -spg  ;# Synopsys Physical Guidance
# Uses floorplan info for wire delay estimation
# Much better correlation with post-P&R timing
```

### 3.6 Path Groups

```tcl
# Path groups prioritize optimization effort on specific paths
group_path -name reg2reg_core -from [get_clocks core_clk] -to [get_clocks core_clk]
group_path -name io_in -from [get_ports *] -to [get_clocks core_clk]
group_path -name io_out -from [get_clocks core_clk] -to [get_ports *]
group_path -name memory_paths -to [get_cells mem_ctrl/*]

# Critical range: optimize all paths within N ns of the worst path
set_critical_range 0.3 [current_design]
# Optimizes paths with slack < (worst_slack + 0.3ns)
# Without this, tool only focuses on the single worst path
```

---

## 4. Technology Mapping

### 4.1 Library Cells

Standard cell naming conventions:

```
  Cell name format (typical):
    <function><inputs>_<drive_strength>

  Examples:
    AND2_X1  → 2-input AND, drive strength 1x
    AND2_X4  → 2-input AND, drive strength 4x (4x wider transistors)
    NAND3_X2 → 3-input NAND, drive strength 2x
    DFFR_X1  → D flip-flop with reset, 1x drive
    AO22_X1  → AND-OR: (a&b)|(c&d), 1x drive

  Vt (threshold voltage) flavors:
    HVT  → High Vt:    SLOWEST,  LEAST leakage,  most area-efficient
    SVT  → Standard Vt: MODERATE, moderate leakage
    LVT  → Low Vt:     FAST,     HIGH leakage
    ULVT → Ultra-Low Vt: FASTEST, HIGHEST leakage

  Speed comparison (7nm example, inverter delay):
    HVT:  ~25 ps
    SVT:  ~18 ps
    LVT:  ~13 ps
    ULVT: ~10 ps

  Leakage comparison:
    HVT:  1x     (baseline)
    SVT:  3-5x
    LVT:  10-20x
    ULVT: 30-50x
```

### 4.2 Cell Selection Trade-offs

```
  ┌──────────────────┬──────────┬──────────┬──────────┬──────────┐
  │ Property         │ X1 (min) │ X2       │ X4       │ X8 (max) │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Drive strength   │ Low      │ Medium   │ High     │ Highest  │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Speed (unloaded) │ Slowest  │ Moderate │ Fast     │ Fastest  │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Area             │ Smallest │ 2x       │ 4x       │ 8x       │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Leakage power    │ Lowest   │ 2x       │ 4x       │ 8x       │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ Input cap        │ Lowest   │ 2x       │ 4x       │ 8x       │
  ├──────────────────┼──────────┼──────────┼──────────┼──────────┤
  │ When to use      │ Off-crit │ Default  │ Medium   │ High     │
  │                  │ paths    │ mapping  │ fanout   │ fanout   │
  │                  │          │          │ or crit  │ buffers  │
  └──────────────────┴──────────┴──────────┴──────────┴──────────┘

  Key insight: Larger drive strength helps with heavily loaded nets
  but the cell itself is slower for lightly loaded nets (larger
  internal capacitance). There is an OPTIMAL drive strength for
  each fanout/load scenario.
```

### 4.3 Mapping Algorithms

**DAG Covering:**

```
  Subject graph (Boolean network) is a DAG (directed acyclic graph).
  Library cells are represented as patterns (sub-DAGs).
  Mapping = covering every node in subject graph with library patterns.
  
  Goal: minimize area (or delay) while covering all nodes.

  Example:
    Subject: a AND b AND c  (a chain of 2 AND2 gates)
    
    Option A: AND2(a,b) → n1; AND2(n1,c) → out
              Cost: 2 cells
    
    Option B: AND3(a,b,c) → out  (if AND3 exists in library)
              Cost: 1 cell, faster (fewer stages)
```

**Boolean Matching (NPN equivalence):**

Two functions are NPN-equivalent if one can be transformed to the other by:
- **N**egation of inputs
- **P**ermutation of inputs
- **N**egation of output

```
  Example: NAND2(a,b) = NOT(AND(a,b))
           NOR2(a,b)  = NOT(OR(a,b)) = NOT(AND(NOT(a),NOT(b)))
           
  NAND2 and NOR2 are NPN-equivalent:
    NAND2(a,b) → negate both inputs → NOR2(a,b)
    (DeMorgan's: NAND(a,b) = OR(~a,~b) = NOR after input negation)
  
  This means: if a function matches NAND2 with some input inversions,
  the tool can use NOR2 + inverters (or vice versa) depending on
  what's cheaper in the library.
```

### 4.4 Multi-Vt Optimization

```
  Strategy: Start all-HVT, swap to lower Vt on critical paths only.

  Flow:
    1. First compile with target_library = HVT only
       → All cells are HVT, minimum leakage
       → Many timing violations expected

    2. compile_ultra or optimize:
       → Tool swaps cells on critical paths to SVT, then LVT
       → Each swap: gain speed, pay with leakage

    3. Result: ~70-80% cells remain HVT, ~15-25% SVT, ~5% LVT

  Leakage comparison:
    All-LVT design:  1.0 mW leakage (baseline)
    Multi-Vt design:  0.2 mW leakage (5x reduction!)
    All-HVT design:  0.05 mW leakage (but can't meet timing)
```

```tcl
# In DC:
set_multi_vth_constraint -lvth_groups {LVT} -lvth_percentage 10
# Limit LVT usage to max 10% of cells

# Or use multiple libraries:
set_app_var target_library "hvt.db svt.db lvt.db"
compile_ultra
# Tool automatically picks optimal Vt per cell
```

---

## 5. Timing Closure Methodology

### 5.1 The Iterative Loop

```
  ┌──────────────────────────────┐
  │  Write/Refine SDC Constraints│ <──────────┐
  └──────────────┬───────────────┘            │
                 │                             │
                 v                             │
  ┌──────────────────────────────┐            │
  │  Run Synthesis (compile_ultra)│            │
  └──────────────┬───────────────┘            │
                 │                             │
                 v                             │
  ┌──────────────────────────────┐            │
  │  Analyze Timing Reports       │            │
  │  (report_timing, report_qor)  │            │
  └──────────────┬───────────────┘            │
                 │                             │
           ┌─────┴──────┐                     │
           │  Timing    │── NO ──> Fix ────────┘
           │  Met?      │         (restructure RTL,
           └─────┬──────┘          adjust constraints,
                 │ YES              add path groups,
                 v                  retime, pipeline)
  ┌──────────────────────────────┐
  │  Proceed to P&R               │
  └──────────────────────────────┘
```

### 5.2 Reading Timing Reports

```
  Example DC timing report:

  Startpoint: cpu/alu/reg_a (rising edge-triggered flip-flop clocked by core_clk)
  Endpoint:   cpu/wb/reg_result (rising edge-triggered flip-flop clocked by core_clk)
  Path Group: core_clk
  Path Type:  max (setup check)

  Point                                    Incr       Path
  ──────────────────────────────────────────────────────────
  clock core_clk (rise edge)               0.000      0.000
  clock network delay (ideal)              0.600      0.600
  cpu/alu/reg_a/CK (DFFR_X1)              0.000      0.600
  cpu/alu/reg_a/Q (DFFR_X1)               0.120      0.720  ← CK-to-Q delay
  cpu/alu/u_add/A[0] (AND2_X1)            0.015      0.735  ← wire delay
  cpu/alu/u_add/Y (AND2_X1)               0.085      0.820  ← cell delay
  cpu/alu/u_add2/A[0] (FA_X1)             0.010      0.830
  cpu/alu/u_add2/CO (FA_X1)               0.130      0.960  ← carry chain!
  ... (carry propagation through 32 stages) ...
  cpu/alu/u_add32/S (FA_X1)               0.095      2.850
  cpu/wb/mux_result/Y (MUX2_X1)           0.075      2.925
  cpu/wb/reg_result/D (DFFR_X1)           0.010      2.935  ← data arrival
  data arrival time                                   2.935

  clock core_clk (rise edge)               1.000      1.000  ← period
  clock network delay (ideal)              0.600      1.600
  clock uncertainty                       -0.150      1.450
  cpu/wb/reg_result/CK (DFFR_X1)          0.000      1.450
  library setup time                      -0.050      1.400
  data required time                                  1.400
  ──────────────────────────────────────────────────────────
  slack (MET/VIOLATED)                               -1.535  ← VIOLATED!

  Analysis:
    Data arrives at 2.935 ns
    Data required by 1.400 ns (1.0ns period + 0.6ns clk delay - 0.15 uncert - 0.05 setup)
    Slack = 1.400 - 2.935 = -1.535 ns (violated by 1.535 ns!)
    
  Root cause: 32-bit carry chain through ripple carry adder
  Fix: Use carry-lookahead or Kogge-Stone adder (datapath optimization)
       Or pipeline the adder into 2 stages
```

### 5.3 Common Timing Closure Techniques

```
  ┌───────────────────────┬────────────────────────────────────────┐
  │ Technique             │ When to Use                            │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Pipelining            │ Large combo delay, can add latency     │
  │                       │ (adders, multipliers, long paths)      │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Register balancing    │ Uneven pipeline stages (retiming)      │
  │                       │ compile_ultra -retime                  │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Logic restructuring   │ Deep logic cones, rewrite for less     │
  │                       │ depth (e.g., tree vs chain structure)  │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Architecture change   │ Fundamental bottleneck (e.g., change   │
  │                       │ from ripple to CLA adder in RTL)       │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Path group + weight   │ Tool not focusing on right paths       │
  │                       │ group_path -weight 5.0                 │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Critical range        │ Tool fixes WNS but many near-critical  │
  │                       │ set_critical_range 0.2                 │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Multicycle path       │ Path doesn't need single-cycle timing  │
  │                       │ (config registers, slow interfaces)    │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Ungroup               │ Boundary preventing cross-module opt   │
  ├───────────────────────┼────────────────────────────────────────┤
  │ Cell sizing (manual)  │ Post-compile, upsize specific cells    │
  │                       │ size_cell cpu/alu/u1 AND2_X4           │
  └───────────────────────┴────────────────────────────────────────┘
```

### 5.4 Incremental Compile

```tcl
# When to use incremental compile:
# - After initial compile_ultra achieves close-to-met timing
# - After adjusting constraints or adding path groups
# - NOT for major timing violations (> 20% of period)

compile_ultra -incremental
# Performs:
#   - Local cell resizing
#   - Buffer insertion/removal
#   - Pin swapping
#   - Gate cloning for fanout
# Does NOT:
#   - Restructure logic
#   - Remap large sections
#   - Ungroup
```

### 5.5 Hold Violation Fixing at Synthesis

```tcl
# Tell synthesis to fix hold violations
set_fix_hold [get_clocks core_clk]
# Tool inserts delay buffers on short paths

# CAUTION: Fixing hold at synthesis is approximate because
# actual clock skew is unknown pre-CTS.
# Post-CTS hold fixing (in P&R) is more accurate.
# Some teams intentionally skip hold fixing at synthesis.
```

### 5.6 Topographical Mode (DCT / DC-Topo)

```tcl
# Placement-aware synthesis for better PPA correlation
# In DC:
create_floorplan -core_utilization 0.7 -core_aspect_ratio 1.0
compile_ultra -spg  ;# Synopsys Physical Guidance

# Benefits:
# - Wire delay estimation based on actual placement (not wireload models)
# - Better correlation with post-P&R timing (within 5-10%)
# - Reduces ECO iterations between synthesis and P&R
# - Identifies congestion hotspots early

# Traditional (non-topo) synthesis uses wireload models:
# set_wire_load_model -name "medium" -library lib_name
# These are statistical estimates -- poor accuracy at advanced nodes
```

---

## 6. Area Optimization

### 6.1 Area Recovery After Timing Closure

```tcl
# After timing is met, recover area on non-critical paths:

# 1. Downsize cells with large positive slack
compile_ultra -incremental -area

# 2. Report area breakdown
report_area -hierarchy
report_reference -hierarchy  ;# shows cell type usage

# 3. Analyze specific cell usage
report_reference -by_type
#  Example output:
#  Reference    Count   Area    
#  AND2_X1      15234   30468   
#  AND2_X4       2340   18720   ← can some X4s become X1s?
#  DFFR_X1      45000   450000  
#  BUF_X8        3200   51200   ← hold buffers, are all needed?
```

### 6.2 Sequential Optimization

**Register merging:** Combine duplicate registers (same logic driving same
D input) into one register.

```
  Before:
    [comb_logic] ──> reg_a/D     reg_a/Q ──> [fanout1]
    [comb_logic] ──> reg_b/D     reg_b/Q ──> [fanout2]
    (reg_a and reg_b have identical D inputs)

  After merging:
    [comb_logic] ──> reg_a/D     reg_a/Q ──> [fanout1]
                                 reg_a/Q ──> [fanout2]
    (reg_b removed, its fanout driven by reg_a)
    Savings: 1 flip-flop
```

**Clock gating inference:**

```tcl
# Synthesis infers clock gating from RTL enable patterns
# RTL:
#   always @(posedge clk)
#     if (enable)
#       q <= d;
#
# Synthesizes to:
#   ICG (Integrated Clock Gating cell):
#     gated_clk = clk & enable (with latch to avoid glitch)
#   DFF with gated_clk instead of clk

# In DC:
set_clock_gating_style -sequential_cell latch \
    -minimum_bitwidth 4
# Only gate groups of 4+ FFs (overhead of ICG cell not worth it for fewer)
compile_ultra -gate_clock

# Report:
report_clock_gating
# Shows: number of ICG cells, number of gated registers, coverage %
```

### 6.3 Unused Logic Removal

```tcl
# DC automatically removes:
# - Unloaded nets (output goes nowhere)
# - Unloaded registers
# - Constants propagated through logic

# To prevent removal of specific logic:
set_dont_touch [get_cells debug_logic/*]
# Preserves cells for debug even if synthesis thinks they're unused
```

---

## 7. Power Optimization in Synthesis

### 7.1 Clock Gating Insertion

```
  Without clock gating:
  ═══════════════════════

  CLK ──────────────────────────> all FFs always clocked
  Power: C * V^2 * f * alpha  (alpha = activity factor ≈ 0.1-0.5)
  Even when FFs hold the same value, clock tree toggles = wasted power

  With clock gating:
  ═══════════════════

  CLK ──> [ICG] ──> gated_clk ──> group of FFs
              ^
           enable

  When enable=0: gated_clk = 0 (static), no switching power
  Savings: proportional to (1 - duty_cycle_of_enable)

  ICG cell structure (latch-based):
  
  enable ──> [Latch (transparent when CLK=0)] ──> AND ──> gated_clk
                                                    ^
  CLK ────────────────────────────────────────────┘

  Latch prevents glitch: enable sampled when CLK is LOW,
  so gated_clk rises cleanly with CLK.
```

**RTL coding for clock gating inference:**

```verilog
// GOOD: Tool infers clock gating
always @(posedge clk) begin
    if (enable)
        data_out <= data_in;
    // implicit else: data_out retains (FF holds value)
end

// BAD: Tool cannot infer clock gating (explicit else with assignment)
always @(posedge clk) begin
    if (enable)
        data_out <= data_in;
    else
        data_out <= data_out;  // self-assignment prevents gating inference
end
// (some tools handle this, but it's not guaranteed)
```

### 7.2 Operand Isolation

```
  Without operand isolation:
    [MUL] computes a*b EVERY cycle, even when result is unused
    (MUL inputs toggle, consuming dynamic power)

  With operand isolation:
    When result not needed: AND gates force MUL inputs to 0
    → MUL inputs don't toggle → no dynamic power in MUL
    
    ┌─────┐
  a ─┤ AND ├──> [MUL] ──> result
    │     │         
  en ┤     │
    └─────┘

    en=0: MUL inputs = 0 (static, no toggling)
    en=1: MUL inputs = a (normal operation)
```

```tcl
# In DC:
set_operand_isolation_style -logic and
compile_ultra -gate_clock  ;# also enables operand isolation
```

### 7.3 Multi-Vt Swap for Leakage Reduction

```tcl
# Post-compile leakage optimization:
# Swap non-critical cells from LVT → SVT → HVT

# In DC:
set_max_leakage_power 0.5  ;# target in mW
compile_ultra

# Or post-compile:
optimize_netlist -area  ;# includes leakage optimization

# In Genus:
set_db design:top .max_leakage_power 500  ;# in uW
syn_opt
```

### 7.4 Power-Aware Compile with SAIF

```tcl
# SAIF (Switching Activity Interchange Format) provides
# actual signal toggle rates from simulation

# 1. Run RTL simulation, dump SAIF
# (in simulator: $set_toggle_region("top"); ... $report_toggle;)

# 2. Back-annotate in synthesis
read_saif -input simulation.saif -instance top

# 3. Power-aware compile
set_max_dynamic_power 100  ;# mW
set_max_leakage_power 1    ;# mW
compile_ultra -gate_clock

# 4. Report power
report_power -hierarchy -verbose
#  Example output:
#  Hierarchy         Internal  Switching  Leakage  Total
#  ─────────────────────────────────────────────────────
#  top                25.3 mW   18.7 mW   1.2 mW  45.2 mW
#    cpu_core          15.1 mW  10.2 mW   0.8 mW  26.1 mW
#      alu              8.3 mW   5.1 mW   0.3 mW  13.7 mW
#      reg_file         3.2 mW   2.8 mW   0.2 mW   6.2 mW
#    mem_ctrl           6.4 mW   5.1 mW   0.2 mW  11.7 mW
#    periph             3.8 mW   3.4 mW   0.2 mW   7.4 mW
```

**Without SAIF:** Tool assumes default toggle rate (typically 0.1 or 10%)
for all signals -- very inaccurate. **With SAIF:** Actual toggle rates from
realistic simulation -- much more accurate power optimization targeting.

---

## 8. Design Compiler vs Genus Comparison

### 8.1 Command Mapping Table

```
  ┌────────────────────────────┬────────────────────────────────────┐
  │ Design Compiler (Synopsys) │ Genus (Cadence)                    │
  ├────────────────────────────┼────────────────────────────────────┤
  │ read_verilog               │ read_hdl -sv                       │
  │ analyze -format sverilog   │ read_hdl -sv                       │
  │ elaborate                  │ elaborate                          │
  │ link                       │ (automatic after elaborate)        │
  │ current_design             │ (automatic)                        │
  ├────────────────────────────┼────────────────────────────────────┤
  │ source constraints.sdc     │ read_sdc constraints.sdc           │
  │ create_clock               │ create_clock (same SDC)            │
  ├────────────────────────────┼────────────────────────────────────┤
  │ compile_ultra              │ syn_generic + syn_map + syn_opt    │
  │ compile_ultra -incremental │ syn_opt -incr                      │
  │ compile_ultra -retime      │ set_db .retime true; syn_opt       │
  │ compile_ultra -gate_clock  │ (automatic in syn_generic)         │
  │ compile_ultra -spg (topo)  │ set_db .place true (phys synth)    │
  ├────────────────────────────┼────────────────────────────────────┤
  │ set_dont_touch             │ set_db [get_cells ...] .dont_touch true │
  │ set_ungroup                │ set_db [get_designs ..] .ungroup true   │
  │ set_boundary_optimization  │ set_db .boundary_opto true         │
  ├────────────────────────────┼────────────────────────────────────┤
  │ report_timing              │ report_timing                      │
  │ report_area                │ report_area                        │
  │ report_power               │ report_power                       │
  │ report_qor                 │ report_qor                         │
  │ report_constraint          │ report_timing -lint                │
  ├────────────────────────────┼────────────────────────────────────┤
  │ write -format verilog      │ write_hdl                          │
  │ write_sdc                  │ write_sdc                          │
  │ write -format ddc          │ write_design -innovus              │
  ├────────────────────────────┼────────────────────────────────────┤
  │ set_app_var target_library │ set_db init_lib_search_path        │
  │                            │ set_db library                     │
  └────────────────────────────┴────────────────────────────────────┘
```

### 8.2 Optimization Differences

```
  ┌──────────────────────┬──────────────────────┬──────────────────────┐
  │ Feature              │ Design Compiler      │ Genus                │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Compile model        │ Single command        │ Three-step           │
  │                      │ (compile_ultra)      │ (generic/map/opt)    │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Datapath opt         │ DesignWare libs      │ Built-in datapath    │
  │                      │ (extensive IP)       │ synthesis            │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Physical awareness   │ DCT/DC-Topo (-spg)   │ Physical synthesis   │
  │                      │                      │ (Genus + Innovus)    │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Multi-Vt handling    │ Excellent, mature     │ Very good            │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Runtime              │ Moderate              │ Generally faster     │
  │                      │                      │ (especially large    │
  │                      │                      │  designs)            │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ P&R integration      │ ICC2 (Synopsys flow) │ Innovus (Cadence     │
  │                      │                      │  flow)               │
  ├──────────────────────┼──────────────────────┼──────────────────────┤
  │ Market position      │ Dominant in ASIC      │ Growing adoption,    │
  │ (as of 2025)         │ synthesis             │ strong in advanced   │
  │                      │                      │ nodes                │
  └──────────────────────┴──────────────────────┴──────────────────────┘
```

### 8.3 When Each Is Preferred

**Design Compiler (DC):**
- Mature, established flow with extensive documentation
- Best-in-class DesignWare library for complex datapaths
- Dominant tool -- most foundry reference flows use DC
- Required when downstream P&R is ICC2 (Synopsys full flow)

**Genus:**
- Often faster runtime, especially for large designs (> 5M instances)
- Three-step compile gives more control over optimization phases
- Native integration with Innovus P&R (Cadence full flow)
- Competitive or better QoR at advanced nodes (7nm, 5nm, 3nm)
- Growing adoption at major semiconductor companies

**Reality:** Many companies license BOTH tools and benchmark regularly.
The "better" tool depends on the specific design, library, and constraints.
Senior engineers should be proficient in both.

---

## 9. Interview Q&A

### Q1: Walk me through the synthesis flow from RTL to gates.

**A:** Synthesis has three phases. (1) **Translation**: The tool parses RTL
(Verilog/VHDL) into a technology-independent Boolean network using generic
(GTECH) cells. It infers flip-flops, latches, and memories from the coding
style. (2) **Optimization**: Boolean and structural optimization including
factoring, decomposition, retiming, resource sharing, and datapath
optimization. All done at the GTECH level. (3) **Technology Mapping**: The
optimized Boolean network is mapped to the target library cells using DAG
covering and Boolean matching algorithms. Cell selection considers timing
(pick faster cells on critical paths), area (smallest cells off critical),
and DRV constraints (max transition, capacitance, fanout). The output is a
gate-level netlist.

### Q2: What is the difference between set_false_path and set_clock_groups -asynchronous?

**A:** `set_false_path -from clk_a -to clk_b` removes timing analysis from
clk_a to clk_b only. You need a second command for clk_b to clk_a.
`set_clock_groups -asynchronous -group clk_a -group clk_b` is bidirectional --
it removes timing in BOTH directions. Additionally, set_clock_groups affects
SI/crosstalk analysis (the tool knows both clocks are physically present),
while set_false_path does not convey this information. For truly asynchronous
clock domains, set_clock_groups is the correct and preferred approach.

### Q3: Explain set_multicycle_path with a concrete example and the hold adjustment.

**A:** A multicycle path of N means data is valid for N clock cycles. For
example, a configuration register written every 4th cycle:
```
set_multicycle_path 4 -setup -from config_reg -to core_reg
set_multicycle_path 3 -hold  -from config_reg -to core_reg
```
The setup check moves from launch+T to launch+4T, giving 4x timing budget.
The hold check (N-1=3) moves the hold reference to launch+(4-3)T = launch+T,
checking that old data doesn't corrupt capture one cycle after launch. Without
the hold adjustment, the hold check would be at launch+4T-0 = launch+4T, which
is overly pessimistic and would require huge insertion delays. The standard
practice is always to pair `-setup N` with `-hold (N-1)`.

### Q4: What is the difference between -physically_exclusive and -logically_exclusive in set_clock_groups?

**A:** Physically exclusive clocks CANNOT coexist on silicon (e.g., test_clk
and func_clk through the same MUX -- only one propagates). The tool can share
CTS resources between them since they never conflict. Logically exclusive
clocks DO coexist physically (both clock trees are built) but are functionally
mutually exclusive (controlled by a mode register). The tool must build
independent clock trees. Both remove timing analysis between the groups, but
the physical implications for CTS, power analysis, and SI are different.

### Q5: How does retiming work and what are its limitations?

**A:** Retiming moves registers forward or backward across combinational logic
to balance pipeline stage delays. Forward retiming moves a register from before
a logic block to after it; backward retiming does the reverse. The tool
verifies functional equivalence and adjusts register count as needed (at fanout
points, forward retiming may duplicate registers; at reconvergent points,
backward retiming may merge them). Limitations: cannot retime across I/O
boundaries (changes interface timing), clock domain boundaries (changes CDC
behavior), or registers with reset/initial values (would change reset state).
Also cannot retime if register has dont_touch or scan attributes.

### Q6: Explain the impact of clock gating on power and area.

**A:** Clock gating eliminates switching power on the clock tree and flip-flops
when the enable is inactive. In a typical SoC, the clock tree consumes 30-50%
of total dynamic power, so gating can save 20-40% dynamic power. Area overhead
is one ICG cell per group of gated FFs (typically 4-32 FFs per ICG). The ICG
cell contains a latch (to prevent glitches) and an AND gate. It adds ~1 gate
equivalent per group. The break-even point is typically 3-4 FFs: gating fewer
FFs wastes area; gating more FFs saves net area by potentially reducing clock
tree buffering. Set minimum bitwidth with `set_clock_gating_style
-minimum_bitwidth 4`.

### Q7: What is boundary optimization and when would you disable it?

**A:** Boundary optimization allows the synthesis tool to propagate constants,
remove unused ports, and merge duplicate logic across hierarchical boundaries.
For example, if a port is tied to constant 0, the tool can propagate this into
the sub-module and simplify the logic. Disable it when: (1) You need to
preserve the exact port interface for ECO or late integration. (2) The
sub-module is shared (instantiated multiple times) and boundary optimization
would specialize it differently for each instance. (3) You need to match a
reference netlist for formal verification against a specific hierarchy.

### Q8: How do you constrain a DDR interface in SDC?

**A:** DDR interfaces transfer data on both rising and falling clock edges.
You need input/output delays referenced to both edges:
```
set_input_delay -clock ddr_clk -max 0.8 [get_ports data]
set_input_delay -clock ddr_clk -max 0.8 -clock_fall -add_delay [get_ports data]
```
The `-clock_fall` references the falling edge, and `-add_delay` prevents the
second constraint from overwriting the first. Both max and min delays should be
specified for setup and hold analysis. The generated clock for DDR typically has
non-50% duty cycle or phase shift that must be accurately modeled.

### Q9: What is the difference between compile_ultra and compile_ultra -incremental?

**A:** `compile_ultra` performs full optimization: ungrouping, Boolean
restructuring, datapath optimization, technology mapping, and timing/area/power
optimization from scratch. `compile_ultra -incremental` performs only local,
non-destructive optimizations: cell resizing, buffer insertion/removal, pin
swapping, and gate cloning. It preserves the overall structure. Use incremental
after: adjusting constraints and re-optimizing, adding path groups, or minor
ECO changes. Do NOT use incremental when the design has major timing violations
requiring structural changes.

### Q10: How do you handle max_delay constraints for CDC paths?

**A:** Use `set_max_delay -datapath_only` for CDC paths. The `-datapath_only`
flag is critical -- it constrains only the combinational data path delay between
the source and destination registers, excluding clock tree delays. Without this
flag, the tool would include clock path delays in the check, which is
meaningless for asynchronous crossings (there's no defined phase relationship).
The max_delay value should be set to ensure the CDC data arrives within the
reconvergence window expected by the synchronizer. Typical value: 1-2 clock
periods of the destination domain.

### Q11: What is wire load model and why is it obsolete?

**A:** Wire load models (WLM) are statistical lookup tables that estimate wire
capacitance and resistance based on net fanout. They were used in synthesis when
physical information wasn't available. They're largely obsolete at advanced
nodes (< 28nm) because: (1) Wire delay is a dominant portion of total delay
and statistical estimation is too inaccurate. (2) Topographical synthesis
(DCT/DC-Topo or Genus physical mode) uses actual placement data for wire
estimation. (3) Post-P&R timing can differ by 30-50% from WLM-based synthesis.
Modern flows use physical-aware synthesis or directly feed floorplan data.

### Q12: Explain the concept of critical range and its importance.

**A:** Critical range defines how deep below the worst negative slack the tool
should optimize. If WNS = -0.5ns and critical range = 0.3ns, the tool
optimizes all paths with slack between -0.5ns and -0.2ns. Without critical
range, the tool focuses only on the single worst path, which may fix WNS but
create new violations as resources are pulled from near-critical paths. Setting
appropriate critical range (typically 10-20% of clock period) distributes
optimization effort across all nearly-critical paths, resulting in better
overall TNS (Total Negative Slack) and faster convergence.

### Q13: What are Design Rule Violations (DRV) and why do they matter?

**A:** DRVs include max_transition (signal slew too slow), max_capacitance
(output overloaded), and max_fanout (too many loads). They matter because:
(1) Excessive transition time causes increased short-circuit power and can
violate library characterization ranges (making timing analysis inaccurate).
(2) Excessive capacitance stresses drivers and degrades reliability (EM).
(3) High fanout creates long-wire routing problems in P&R. The tool fixes
DRVs by inserting buffers, upsizing drivers, or restructuring logic. DRV
fixing can compete with timing optimization -- sometimes fixing a DRV worsens
timing on a critical path.

### Q14: How does multi-Vt optimization work?

**A:** The library provides cells in multiple threshold voltage variants:
HVT (slow, low leakage), SVT (moderate), LVT (fast, high leakage), and
sometimes ULVT. The tool starts with all-HVT (minimum leakage) and selectively
swaps cells on timing-critical paths to lower Vt for speed. The result is
typically 70-80% HVT, 15-25% SVT, and 5% LVT. This achieves 3-5x leakage
reduction compared to all-LVT while meeting the same timing. The
`set_multi_vth_constraint` command can limit the percentage of LVT cells,
preventing the optimizer from overusing fast-but-leaky cells.

### Q15: What is SAIF and how does it improve power optimization?

**A:** SAIF (Switching Activity Interchange Format) captures signal toggle rates
and static probabilities from RTL or gate-level simulation. Without SAIF,
synthesis assumes a default activity factor (typically 10%) for all signals,
leading to inaccurate power estimation. With SAIF back-annotation, the tool
knows actual signal activities and can: (1) Focus clock gating on
highest-activity registers. (2) Apply operand isolation to frequently-inactive
but power-hungry units. (3) Size cells appropriately for their actual switching
frequency. This typically improves power estimation accuracy from +/-50% to
+/-15% and enables 10-20% additional power optimization.

### Q16: When should you ungroup a module and when should you not?

**A:** Ungroup when: the module is on a critical timing path and hierarchy
boundaries prevent cross-module optimization; the module is small glue logic;
the module is a wrapper with no meaningful hierarchy. Do NOT ungroup when:
the module is large (flattening makes optimization intractable -- DC may run
out of memory); you need the hierarchy for ECO insertion; the module is
instantiated multiple times and needs to be preserved as a single optimized
entity; the module is a hard macro or IP. A good practice: ungroup small
modules (<5K gates) on critical paths, keep large modules (>50K gates) grouped.

### Q17: Explain the timing impact of source latency vs network latency.

**A:** Source latency is the delay from the actual clock source (e.g.,
oscillator) to the clock definition point (e.g., PLL input or chip clock
port). Network latency is the delay from the clock definition point through
the clock tree to flip-flop clock pins. Pre-CTS, network latency is estimated
(set by the user); post-CTS, it's replaced by actual propagated delay. Source
latency persists in both pre- and post-CTS analysis because it's external to
the clock tree. The distinction matters for IO timing: set_input_delay and
set_output_delay reference the clock definition point, so source latency
affects when the external clock edge "actually" arrives.

### Q18: What is operand isolation and when is it beneficial?

**A:** Operand isolation gates the inputs of arithmetic blocks (multipliers,
adders, ALUs) to zero when their output is not needed. This prevents
unnecessary toggling inside the block, saving dynamic power. It's beneficial
when: (1) The block is large (multiplier, divider) with significant internal
switching. (2) The block is inactive a significant fraction of cycles (>30%).
(3) The gating logic (AND gates on inputs) overhead is small relative to the
block. It's NOT beneficial for small blocks or blocks that are active nearly
every cycle. The area overhead is one AND gate per input bit.

### Q19: How do you handle timing closure when synthesis and P&R don't correlate?

**A:** Poor correlation means synthesis timing is optimistic relative to P&R.
Solutions: (1) Use topographical synthesis (DCT -spg or Genus physical mode)
with the actual floorplan -- this models wire delays based on placement,
improving correlation to within 5-10%. (2) Back-annotate parasitics from an
initial P&R run and re-synthesize with realistic wire delays. (3) Over-constrain
synthesis (tighten clock period by 10-15%, increase uncertainty) so that
synthesis timing has margin for P&R degradation. (4) Use consistent libraries
and operating conditions between synthesis and P&R. (5) Ensure DRV constraints
are tight enough to prevent P&R from needing excessive buffering.

### Q20: What is the purpose of group_path and how does it affect optimization?

**A:** `group_path` creates named path groups that the optimizer treats as
independent optimization targets. Without path groups, the tool's global
cost function may sacrifice one path group for another. With explicit groups:
(1) Each group gets dedicated optimization effort. (2) You can assign weights
(-weight 2.0) to prioritize certain groups. (3) You can see per-group WNS/TNS
in reports. Common groups: reg2reg, input-to-reg, reg-to-output, memory paths.
This is especially important when different path groups have very different
timing characteristics -- the tool won't "steal" cells from one group to help
another.

### Q21: Explain the three steps in Genus synthesis (syn_generic, syn_map, syn_opt).

**A:** (1) **syn_generic**: Technology-independent optimization. Performs
Boolean optimization, resource sharing, operator merging, clock gating
inference, and structural transformations. Output is a GTECH netlist.
(2) **syn_map**: Maps the generic netlist to target library cells. Performs
DAG covering, Boolean matching, cell selection for timing/area/power, and
initial DRV fixing. Output is a technology-mapped netlist. (3) **syn_opt**:
Post-mapping optimization. Performs local timing closure (cell sizing, buffer
insertion, pin swapping), hold fixing, leakage optimization, and DRV cleanup.
This three-step approach gives users control -- you can iterate on syn_opt
without re-running syn_generic and syn_map.

### Q22: How do you write RTL that synthesizes efficiently?

**A:** Key guidelines: (1) Use synchronous resets (async resets add logic to
every FF). (2) Avoid latches (use edge-triggered FFs). (3) Code enable
patterns so clock gating is inferred (`if (en) q <= d;` without explicit
else). (4) Avoid very wide case statements with many overlapping conditions.
(5) Keep arithmetic expressions simple and let the tool optimize (don't
manually instantiate adder structures unless absolutely necessary). (6) Use
named `generate` blocks for parameterized code. (7) Avoid combinational loops.
(8) Minimize the use of `casex`/`casez` (can infer X-propagation issues).
(9) Register all module outputs for better boundary optimization. (10) Use
consistent coding styles (always_ff, always_comb in SystemVerilog) for
reliable inference.

### Q23: What happens when you run compile_ultra with -retime?

**A:** The tool performs adaptive retiming: it identifies pipeline stages with
unbalanced combinational delays and moves registers to equalize them. For
example, if stage 1 has 1.5ns combo delay and stage 2 has 0.5ns, retiming
moves logic from stage 1 into stage 2, potentially achieving 1.0ns each. The
tool must verify: (1) Functional equivalence is preserved. (2) I/O timing
is not affected (registers at I/O boundaries are not moved). (3) Clock domain
crossings are not disrupted. (4) Reset behavior is preserved. Retiming can
improve maximum frequency by 10-30% without RTL changes, but it changes the
register locations, which can complicate debug and ECO.

### Q24: What is the difference between set_max_delay and set_multicycle_path?

**A:** `set_multicycle_path N` moves the capture edge to N cycles after launch,
and the hold edge accordingly. It maintains the relationship with the clock
structure. `set_max_delay` specifies an absolute maximum delay in nanoseconds,
independent of clock period. Use multicycle for synchronous paths where data
is valid for multiple cycles. Use max_delay for asynchronous CDC paths (with
`-datapath_only`) where there's no meaningful clock relationship. Do NOT use
max_delay as a substitute for multicycle on synchronous paths -- it bypasses
the clock-based timing framework and can lead to incorrect hold analysis.

---

*End of Logic Synthesis and Optimization Deep Dive*
