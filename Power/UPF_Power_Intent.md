# UPF (Unified Power Format) Power Intent Specification

## 1. Why UPF Exists

RTL (Verilog/VHDL) describes **functional behavior** only. It says nothing about:
- Which blocks can be powered off
- What voltage each block runs at
- What happens to outputs when a block is off
- Which registers retain state across power-down
- How power supplies are switched and sequenced

Without a separate power intent specification, every tool in the flow (synthesis, P&R,
simulation, verification, signoff) would need ad-hoc, inconsistent mechanisms to handle
low-power design. UPF (IEEE 1801) provides a **single, standardized specification** that
is consumed by ALL tools, ensuring consistent power-aware implementation and verification.

**Key principle:** The SAME RTL can be synthesized with DIFFERENT power strategies by
changing only the UPF file. This decouples functional design from power architecture.

### 1.1 UPF vs CPF

| Feature          | UPF (IEEE 1801)            | CPF (Si2/Cadence)            |
|------------------|----------------------------|------------------------------|
| Standard body    | IEEE                       | Si2 (industry consortium)    |
| Primary backer   | Synopsys (originally)      | Cadence (originally)         |
| Current status   | Industry standard          | Largely superseded by UPF    |
| Tool support     | All major EDA tools        | Cadence tools primarily      |
| Language         | TCL-based                  | TCL-based                    |
| Key difference   | Supply sets, successive refinement | Explicit net connections |

**Bottom line:** Use UPF for all new designs. CPF knowledge is useful for legacy projects.

### 1.2 UPF Versions

| Version    | Year | Key Additions                                              |
|------------|------|------------------------------------------------------------|
| UPF 1.0    | 2007 | Core: domains, switches, isolation, retention              |
| UPF 2.0    | 2014 | Supply sets, successive refinement, power states, PST      |
| UPF 2.1    | 2017 | Repeater/buffer strategy, liberty extensions                |
| UPF 3.0    | 2019 | Incremental UPF, improved supply handles, abstract models  |
| UPF 3.1    | 2022 | Refinements for tool interoperability                      |

### 1.3 UPF in the Design Flow

```
  RTL Design (Verilog/VHDL)
      |
      +---- UPF Power Intent ----+
      |                          |
      v                          v
  Synthesis (DC/Genus)       Power-Aware Simulation
      |  (inserts ISO, RET,     (VCS/Xcelium with UPF)
      |   level shifters,       - Verifies power sequences
      |   power switches)       - Outputs X for powered-off blocks
      v                         - Catches cross-domain errors
  Place & Route (ICC2/Innovus)
      |  (places special cells,
      |   routes power grid,
      |   switch insertion)
      v
  Power Analysis (PrimeTime PX / Voltus)
      |  (per-domain power,
      |   IR drop per domain)
      v
  Signoff
```

---

## 2. Core UPF Concepts

### 2.1 Power Domain

A **power domain** is a group of logic instances that share the same power supply behavior.
All cells in a power domain are:
- Powered by the same supply set
- Turned on/off together
- In the same voltage/power state at any given time

Every cell in the design belongs to **exactly one** power domain. The default domain is
the top-level domain (always-on unless explicitly gated).

### 2.2 Supply Network

UPF models power supplies abstractly using:
- **Supply ports:** entry/exit points for power at hierarchy boundaries
- **Supply nets:** wires carrying power within a hierarchy level
- **Supply sets:** abstract bundles of {power, ground} that are assigned to domains

### 2.3 Power States

Each power domain can be in one of several states:
- **ON:** fully powered, normal operation
- **OFF:** power removed, all state lost, outputs undefined
- **RETENTION:** power removed from main logic, but retention supply keeps shadow latches alive

### 2.4 Special Cells

UPF triggers insertion of special cells at domain boundaries and within gated domains:
- **Isolation cells:** clamp outputs of off domains to safe values
- **Level shifters:** convert signal voltage between different-voltage domains
- **Retention registers:** preserve state across power cycling
- **Power switches:** PMOS/NMOS header/footer transistors to gate supply
- **Always-on buffers:** buffers powered by the always-on supply within a gated domain
  (needed to buffer always-on signals that must reach deep into the gated domain)

---

## 3. Complete UPF Tutorial: 3-Domain SoC

### 3.1 Design Architecture

```
  +================================================================+
  |  PD_TOP (Always-On)                                            |
  |  - Bus fabric, interrupt controller, power management unit     |
  |  - Supply: VDD / VSS (always on)                               |
  |                                                                |
  |  +========================+  +============================+   |
  |  |  PD_CPU                |  |  PD_PERI                    |   |
  |  |  - CPU core            |  |  - UART, SPI, GPIO          |   |
  |  |  - Power-gatable       |  |  - Power-gatable            |   |
  |  |  - WITH retention      |  |  - WITHOUT retention        |   |
  |  |  - Supply: VDD_CPU/VSS |  |  - Supply: VDD_PERI/VSS     |   |
  |  +========================+  +============================+   |
  +================================================================+
```

### 3.2 Complete UPF File with Detailed Explanations

```tcl
#================================================================
# UPF Power Intent for 3-Domain SoC
# File: soc_top.upf
#================================================================

# Set the UPF scope to the top-level design
set_scope soc_top

#----------------------------------------------------------------
# STEP 1: Create Power Domains
#----------------------------------------------------------------

# Top-level domain: always-on, contains bus fabric and control logic
# -elements {} means "everything not assigned to another domain"
create_power_domain PD_TOP \
    -elements {.}
# The "." means the current scope (soc_top) and all its direct logic.
# Child domains will carve out their own elements.

# CPU domain: power-gatable with retention
create_power_domain PD_CPU \
    -elements {cpu_core}
# All instances under soc_top/cpu_core belong to PD_CPU

# Peripheral domain: power-gatable without retention
create_power_domain PD_PERI \
    -elements {uart_block spi_block gpio_block}
# Multiple hierarchical instances can belong to one domain

#----------------------------------------------------------------
# STEP 2: Create Supply Ports (chip-level pins)
#----------------------------------------------------------------

# Top-level supply ports (these represent physical package pins)
create_supply_port VDD   -domain PD_TOP -direction in
create_supply_port VSS   -domain PD_TOP -direction in

# Note: VDD_CPU and VDD_PERI are NOT separate package pins in this design.
# They are derived from VDD through power switches.
# If they were separate PMIC rails, they would also be supply ports.

#----------------------------------------------------------------
# STEP 3: Create Supply Nets
#----------------------------------------------------------------

# Always-on supply nets
create_supply_net VDD    -domain PD_TOP
create_supply_net VSS    -domain PD_TOP

# Switched supply nets for gated domains
create_supply_net VDD_CPU  -domain PD_CPU
create_supply_net VDD_PERI -domain PD_PERI

# Ground nets (shared, always-on)
# CPU and PERI share the same ground (header switch design)
create_supply_net VSS    -domain PD_CPU  -reuse
create_supply_net VSS    -domain PD_PERI -reuse
# -reuse: same physical net VSS is shared across domains

# Always-on net within CPU domain (for retention FFs and ISO cells)
create_supply_net VDD_AO_CPU -domain PD_CPU

#----------------------------------------------------------------
# STEP 4: Connect Supply Ports to Supply Nets
#----------------------------------------------------------------

connect_supply_net VDD -ports {VDD}
connect_supply_net VSS -ports {VSS}

#----------------------------------------------------------------
# STEP 5: Create Supply Sets
#----------------------------------------------------------------

# Supply set for the always-on domain
create_supply_set SS_TOP \
    -function {power VDD} \
    -function {ground VSS}

# Supply set for CPU domain (switched power)
create_supply_set SS_CPU \
    -function {power VDD_CPU} \
    -function {ground VSS}

# Supply set for peripheral domain (switched power)
create_supply_set SS_PERI \
    -function {power VDD_PERI} \
    -function {ground VSS}

# Always-on supply set within CPU (for retention/isolation cells)
create_supply_set SS_CPU_AO \
    -function {power VDD} \
    -function {ground VSS}

#----------------------------------------------------------------
# STEP 6: Associate Supply Sets with Domains
#----------------------------------------------------------------

# Each domain gets a primary supply set
associate_supply_set SS_TOP  -handle PD_TOP.primary
associate_supply_set SS_CPU  -handle PD_CPU.primary
associate_supply_set SS_PERI -handle PD_PERI.primary

# CPU domain also needs an always-on supply set (for retention)
associate_supply_set SS_CPU_AO -handle PD_CPU.retention

#----------------------------------------------------------------
# STEP 7: Create Power Switches
#----------------------------------------------------------------

# CPU power switch (header: VDD -> VDD_CPU)
create_power_switch CPU_PSW \
    -domain PD_CPU \
    -input_supply_port  {vin  VDD} \
    -output_supply_port {vout VDD_CPU} \
    -control_port       {sleep_n cpu_sleep_n} \
    -on_state           {on_state  vin {!sleep_n}} \
    -off_state          {off_state {sleep_n}}
# When cpu_sleep_n = 0: switch is ON, VDD_CPU = VDD
# When cpu_sleep_n = 1: switch is OFF, VDD_CPU = floating/0

# Peripheral power switch (header: VDD -> VDD_PERI)
create_power_switch PERI_PSW \
    -domain PD_PERI \
    -input_supply_port  {vin  VDD} \
    -output_supply_port {vout VDD_PERI} \
    -control_port       {sleep_n peri_sleep_n} \
    -on_state           {on_state  vin {!sleep_n}} \
    -off_state          {off_state {sleep_n}}

#----------------------------------------------------------------
# STEP 8: Set Isolation Strategy
#----------------------------------------------------------------

# Isolate CPU domain outputs (when CPU is powered off)
set_isolation CPU_ISO \
    -domain PD_CPU \
    -applies_to outputs \
    -clamp_value 0 \
    -isolation_power_net VDD \
    -isolation_ground_net VSS \
    -name CPU_ISO_STRATEGY

# Isolation control signal for CPU
set_isolation_control CPU_ISO \
    -domain PD_CPU \
    -isolation_signal cpu_iso_en \
    -isolation_sense high \
    -name CPU_ISO_CTRL
# When cpu_iso_en = 1: outputs are clamped to 0
# When cpu_iso_en = 0: outputs pass through normally

# Isolate peripheral domain outputs
set_isolation PERI_ISO \
    -domain PD_PERI \
    -applies_to outputs \
    -clamp_value 0 \
    -isolation_power_net VDD \
    -isolation_ground_net VSS \
    -name PERI_ISO_STRATEGY

set_isolation_control PERI_ISO \
    -domain PD_PERI \
    -isolation_signal peri_iso_en \
    -isolation_sense high \
    -name PERI_ISO_CTRL

#----------------------------------------------------------------
# STEP 9: Set Retention Strategy (CPU domain only)
#----------------------------------------------------------------

# Define which registers in CPU domain retain state
set_retention CPU_RET \
    -domain PD_CPU \
    -retention_power_net VDD \
    -retention_ground_net VSS \
    -name CPU_RET_STRATEGY
# -elements can be used to specify selective retention:
# -elements {cpu_core/reg_file cpu_core/pc cpu_core/csr}
# If -elements is omitted, ALL FFs in the domain get retention

# Retention control signals
set_retention_control CPU_RET \
    -domain PD_CPU \
    -save_signal    {cpu_save high} \
    -restore_signal {cpu_restore high} \
    -name CPU_RET_CTRL
# Save: pulse high before power-down to copy state to shadow latches
# Restore: pulse high after power-up to copy state back from shadow latches

#----------------------------------------------------------------
# STEP 10: Set Level Shifters (for cross-voltage signals)
#----------------------------------------------------------------

# In this design, all domains share the same nominal voltage (VDD).
# Level shifters are still needed if domains can be at different
# voltages due to DVFS or IR drop considerations.

# If CPU runs at a different voltage than TOP:
set_level_shifter CPU_TO_TOP_LS \
    -domain PD_CPU \
    -applies_to outputs \
    -rule both \
    -location self \
    -name CPU_LS
# -rule both: insert for both low-to-high and high-to-low crossings
# -location self: place level shifter in the source domain
# -location parent: place in the receiving domain (alternative)

# Level shifters for signals going INTO CPU from TOP
set_level_shifter TOP_TO_CPU_LS \
    -domain PD_CPU \
    -applies_to inputs \
    -rule both \
    -location self \
    -name CPU_INPUT_LS

# Similarly for peripheral domain
set_level_shifter PERI_TO_TOP_LS \
    -domain PD_PERI \
    -applies_to outputs \
    -rule both \
    -location self \
    -name PERI_LS

#----------------------------------------------------------------
# STEP 11: Define Power States
#----------------------------------------------------------------

# Power states for each supply set
add_power_state SS_TOP \
    -state {TOP_ON  -supply_expr {power == `{FULL_ON, 0.90}}} \
    -state {TOP_OFF -supply_expr {power == `{OFF}}}

add_power_state SS_CPU \
    -state {CPU_ON  -supply_expr {power == `{FULL_ON, 0.90}}} \
    -state {CPU_RET -supply_expr {power == `{OFF}} -simstate RETENTION} \
    -state {CPU_OFF -supply_expr {power == `{OFF}} -simstate CORRUPT}

add_power_state SS_PERI \
    -state {PERI_ON  -supply_expr {power == `{FULL_ON, 0.90}}} \
    -state {PERI_OFF -supply_expr {power == `{OFF}} -simstate CORRUPT}

#----------------------------------------------------------------
# STEP 12: Power State Table (PST)
#----------------------------------------------------------------

# The PST defines LEGAL combinations of domain power states.
# Illegal combinations (e.g., CPU ON but TOP OFF) are flagged as errors.

create_pst SoC_PST \
    -supplies {SS_TOP SS_CPU SS_PERI}

# State: Full Active (all domains on)
add_pst_state FULL_ACTIVE \
    -pst SoC_PST \
    -state {TOP_ON CPU_ON PERI_ON}

# State: CPU Retention (CPU in retention, peripherals on)
add_pst_state CPU_SLEEP \
    -pst SoC_PST \
    -state {TOP_ON CPU_RET PERI_ON}

# State: Peripherals Off (CPU on, peripherals off)
add_pst_state PERI_OFF \
    -pst SoC_PST \
    -state {TOP_ON CPU_ON PERI_OFF}

# State: Deep Sleep (only TOP is on)
add_pst_state DEEP_SLEEP \
    -pst SoC_PST \
    -state {TOP_ON CPU_OFF PERI_OFF}

# State: CPU Retention + Peripherals Off
add_pst_state CPU_RET_PERI_OFF \
    -pst SoC_PST \
    -state {TOP_ON CPU_RET PERI_OFF}

# NOTE: TOP_OFF is never a valid state (always-on domain).
# Any PST entry with TOP_OFF would be illegal.
```

---

## 4. UPF Supply Set Handles -- The Abstract Supply Model

### 4.1 What Are Supply Set Handles?

UPF 2.0 introduced **supply set handles** as an abstraction layer between power domains
and physical supply nets. Instead of directly connecting nets to cells, you assign
abstract "handles" to domains, and the handles resolve to physical nets.

```
  Power Domain
      |
      | has a primary supply set handle (PD.primary)
      | has optional retention supply set handle (PD.retention)
      | has optional isolation supply set handle (PD.isolation)
      |
      v
  Supply Set (abstract bundle)
      |
      | contains: power function -> maps to a supply net
      | contains: ground function -> maps to a supply net
      |
      v
  Supply Net (physical net)
```

### 4.2 Successive Refinement

UPF supports writing power intent at multiple levels of abstraction and refining it
at each design stage:

```
RTL Level (UPF 1):
  - Define power domains and their elements
  - Specify isolation and retention strategies
  - Use abstract supply set handles (no physical net names)

Synthesis Level (UPF 2 - refines UPF 1):
  - Map supply set handles to specific supply nets
  - Specify cell types for isolation, retention, level shifters
  - Add implementation constraints

Physical Design Level (UPF 3 - refines UPF 2):
  - Define physical power switch implementation (daisy chain, ring)
  - Specify power grid design rules per domain
  - Add placement constraints for special cells
```

This allows IP blocks to be delivered with their own "local" UPF that is refined
at the SoC integration level.

### 4.3 Supply Set Handle Resolution Example

```tcl
# IP-level UPF (delivered with IP block)
create_power_domain PD_IP
# The IP uses PD_IP.primary.power and PD_IP.primary.ground
# but does NOT specify what physical nets they map to

# SoC-level UPF (integrator writes this)
create_supply_net VDD_IP_ACTUAL
connect_supply_net VDD_IP_ACTUAL -handle PD_IP.primary.power
connect_supply_net VSS           -handle PD_IP.primary.ground
# Now the abstract handle resolves to a concrete net
```

---

## 5. UPF Scope Rules

### 5.1 How Scope Works

UPF commands apply to the **current scope** set by `set_scope`. Elements within
`create_power_domain -elements` are relative to this scope.

```tcl
set_scope soc_top
create_power_domain PD_CPU -elements {cpu_core}
# Creates domain containing soc_top/cpu_core

set_scope soc_top/cpu_core
create_power_domain PD_CACHE -elements {l1_cache}
# Creates domain containing soc_top/cpu_core/l1_cache
# PD_CACHE is a child of PD_CPU (nested power domain)
```

### 5.2 Hierarchical UPF

For large SoCs, UPF is often written hierarchically:

```tcl
# Top-level UPF
set_scope soc_top
load_upf cpu_core.upf     -scope cpu_core
load_upf periph_block.upf -scope periph_block
# Each sub-UPF defines its own domains, which become children of PD_TOP
```

### 5.3 Scope and Isolation

Isolation is applied at domain boundaries. The `-applies_to` flag controls direction:

```tcl
set_isolation MY_ISO -domain PD_A -applies_to outputs
# Isolates signals going OUT of PD_A to other domains

set_isolation MY_ISO -domain PD_A -applies_to inputs
# Isolates signals coming IN to PD_A from other domains (less common)

set_isolation MY_ISO -domain PD_A -applies_to both
# Isolates both directions
```

**Practical note:** Almost always use `-applies_to outputs`. The receiving domain
is always-on and can handle any input; the sending domain is the one going off and
needs its outputs clamped.

---

## 6. Liberty Extensions for Power-Aware Cells

The `.lib` (Liberty) file must describe special cell attributes so that tools can
correctly use them during synthesis and P&R:

### 6.1 Power Switch Cell

```liberty
cell(HEADER_SWITCH_1X) {
    switch_cell_type : coarse_grain;
    pg_pin(VDD) {
        voltage_name : VDD;
        pg_type : primary_power;
    }
    pg_pin(VVDD) {
        voltage_name : VVDD;
        pg_type : internal_power;
        switch_function : "SLEEP_N";
        /* VVDD = VDD when SLEEP_N=1, floating when SLEEP_N=0 */
    }
    pg_pin(VSS) {
        voltage_name : VSS;
        pg_type : primary_ground;
    }
    pin(SLEEP_N) {
        direction : input;
    }
}
```

### 6.2 Isolation Cell

```liberty
cell(ISO_AND_1X) {
    is_isolation_cell : true;
    dont_use : false;
    pg_pin(VDD) {
        voltage_name : VDD;
        pg_type : primary_power;
    }
    pg_pin(VSS) {
        voltage_name : VSS;
        pg_type : primary_ground;
    }
    pin(A) {
        direction : input;
        isolation_cell_data_pin : true;
    }
    pin(ISO) {
        direction : input;
        isolation_cell_enable_pin : true;
    }
    pin(Z) {
        direction : output;
        function : "A & !ISO";
        power_down_function : "!VDD + VSS";
        /* When VDD is off: Z is undetermined
           When ISO=1: Z = 0 (clamped low) */
    }
}
```

### 6.3 Retention Register

```liberty
cell(DFFR_RET_1X) {
    retention_cell : "ret_pair";
    dont_use : false;
    pg_pin(VDD) {
        voltage_name : VDD;
        pg_type : primary_power;
    }
    pg_pin(VDDAO) {
        voltage_name : VDDAO;
        pg_type : backup_power;
        /* This pin is connected to the always-on supply */
    }
    pg_pin(VSS) {
        voltage_name : VSS;
        pg_type : primary_ground;
    }
    pin(CLK) { ... }
    pin(D)   { ... }
    pin(Q)   { ... }
    pin(SAVE) {
        direction : input;
        retention_pin (save_restore, "1");
        /* SAVE=1: copy master/slave to shadow latch */
    }
    pin(RESTORE) {
        direction : input;
        retention_pin (restore, "1");
        /* RESTORE=1: copy shadow latch back to master/slave */
    }
}
```

### 6.4 Level Shifter Cell

```liberty
cell(LS_LH_1X) {
    is_level_shifter : true;
    level_shifter_type : LH;
    /* LH = low-to-high, HL = high-to-low */
    input_voltage_range(0.5, 0.9);
    output_voltage_range(0.7, 1.1);
    pg_pin(VDDL) {
        voltage_name : VDDL;
        pg_type : primary_power;
    }
    pg_pin(VDDH) {
        voltage_name : VDDH;
        pg_type : primary_power;
    }
    pg_pin(VSS) {
        voltage_name : VSS;
        pg_type : primary_ground;
    }
    pin(A) { direction : input; }
    pin(Z) { direction : output; function : "A"; }
}
```

### 6.5 Always-On Buffer

```liberty
cell(AO_BUF_1X) {
    always_on : true;
    /* This cell is powered by the always-on supply even in a gated domain */
    pg_pin(VDD) {
        voltage_name : VDD;
        pg_type : primary_power;
    }
    pg_pin(VSS) {
        voltage_name : VSS;
        pg_type : primary_ground;
    }
    pin(A) { direction : input;  }
    pin(Z) { direction : output; function : "A"; }
}
```

---

## 7. UPF Verification Flow

### 7.1 Static UPF Checks

Static checks verify UPF structural correctness WITHOUT simulation:

```
Checks performed:
  1. Every output of a gated domain has an isolation cell
     - Missing isolation -> outputs float when domain is off -> corruption
  
  2. Every cross-voltage signal has a level shifter
     - Missing level shifter -> voltage mismatch -> potential functional failure
  
  3. Every power switch has proper control connectivity
     - Dangling control -> switch never turns on/off
  
  4. Retention control signals are connected and properly sequenced
     - Missing save/restore -> retention FFs don't save/restore
  
  5. No combinational paths from off domain to on domain without isolation
     - Even through intermediate logic
  
  6. Supply connectivity: all domains have valid supply sets
  
  7. PST legality: all defined states are consistent
```

**Tool commands:**
```tcl
# Synopsys VC LP (Verification Compiler Low Power)
check_lp -upf soc_top.upf -design soc_top
# Reports: missing isolation, missing level shifters, etc.

# Cadence Conformal LP
read_upf soc_top.upf
check_power_intent
# Similar checks plus equivalence checking with power intent
```

### 7.2 Dynamic UPF Simulation (Power-Aware Simulation)

Power-aware simulation is the most important verification step for low-power designs.
It verifies that:
1. Outputs of powered-off domains become X (unknown)
2. Isolation cells properly clamp outputs before power-off
3. Retention registers correctly save and restore state
4. Power sequencing is correct (no reads from off domains)
5. No functional failures during power transitions

**How it works:**
```
Regular RTL simulation:
  - All signals have defined values (0 or 1) at all times
  - Cannot detect "reading from a powered-off domain" because everything looks functional

Power-aware simulation:
  - When a domain is powered off:
    * All flip-flops in that domain are forced to X
    * All combinational outputs of that domain become X
    * Isolation cells clamp outputs to their specified value (0 or 1)
  - When a domain is powered on:
    * Retention registers restore their saved value (not X)
    * Non-retention registers remain X until reset/initialized
  - Any X propagation to an always-on domain is a BUG
    (means isolation is missing or incorrect)
```

**Tool setup:**
```tcl
# Synopsys VCS with UPF
vcs -upf soc_top.upf -power_top soc_top \
    -f filelist.f -debug_access+all

# Cadence Xcelium with UPF
xrun -upf soc_top.upf -powertop soc_top \
    -f filelist.f -access +rwc

# Mentor Questa with UPF
vlog -f filelist.f
vsim -upf soc_top.upf soc_top
```

### 7.3 Bugs That Power-Aware Simulation Catches

**Bug 1: Missing isolation on a qualified output**
```verilog
// RTL (inside CPU domain):
assign bus_valid = internal_valid & bus_grant;

// If bus_valid is not isolated:
// When CPU is off: bus_valid = X
// This X propagates to the bus arbiter (always-on) -> functional failure

// Fix: ensure UPF covers all outputs, including qualified (AND-gated) ones
```

**Bug 2: Reading from off domain through combinational logic**
```verilog
// In always-on domain:
assign result = cpu_data + peri_data;  // Both from gated domains

// If CPU is off: cpu_data = X (isolated to 0), but adder still propagates
// If isolation value of 0 is not the expected "idle" value -> wrong result
```

**Bug 3: Incorrect restore timing**
```verilog
// Testbench power-up sequence:
#10 peri_sleep_n = 1;    // Turn on power
#5  cpu_restore = 1;      // Restore retention -- TOO EARLY!
// Power might not be stable yet
// Retention restore requires stable voltage to correctly transfer data
```

**Bug 4: Wrong clamp value causing logic inversion**
```verilog
// RTL: reset_n is active-low reset (0 = reset asserted)
// UPF clamps to 0 on isolation -> reset is asserted when CPU is off
// This is CORRECT for reset_n (safe state)

// But if you accidentally clamp an active-HIGH enable to 0:
// enable = 0 when CPU is off -> downstream logic is disabled (maybe correct)
// enable = 0 when CPU wakes up but before isolation deasserted -> extra cycle of disable
// Need to verify the timing of isolation deassertion
```

---

## 8. UPF 2.0/3.0 Enhancements

### 8.1 Supply Expressions (UPF 2.0)

Supply expressions allow defining power states based on supply voltage values:

```tcl
add_power_state SS_CPU \
    -state {TURBO   -supply_expr {power == `{FULL_ON, 1.05}}} \
    -state {NOMINAL -supply_expr {power == `{FULL_ON, 0.90}}} \
    -state {LOW_PWR -supply_expr {power == `{FULL_ON, 0.75}}} \
    -state {RET     -supply_expr {power == `{PARTIAL_ON, 0.50}}} \
    -state {OFF     -supply_expr {power == `{OFF}}}
```

### 8.2 Repeater Strategies (UPF 2.1)

For signals that traverse long distances within a gated domain (e.g., always-on clock
signal buffered through a gated domain), UPF can specify that repeater buffers must
be always-on:

```tcl
set_repeater CPU_AO_REPEATER \
    -domain PD_CPU \
    -applies_to {clk reset_n} \
    -lib_cells {AO_BUF_1X AO_BUF_2X} \
    -name CPU_AO_BUF
```

### 8.3 Abstract Power Model for IP Delivery (UPF 3.0)

IP vendors can deliver an abstract UPF model that specifies:
- What power domains the IP has
- What supplies it needs
- What isolation/retention it requires
- WITHOUT exposing internal implementation details

```tcl
# Abstract UPF for a licensable IP block
create_power_domain PD_IP_TOP -supply {primary}
create_power_domain PD_IP_CORE \
    -supply {primary} \
    -supply {retention}

# The integrator connects these abstract supplies to real nets
# without needing to know the internal structure
```

### 8.4 Incremental UPF (UPF 3.0)

Allows applying UPF modifications without rewriting the entire file:

```tcl
# Base UPF defines the structure
# Incremental UPF adds or overrides specific aspects

# Example: add an extra power state that wasn't in the original
begin_power_model_update
  add_power_state SS_CPU \
      -state {ULTRA_LOW -supply_expr {power == `{FULL_ON, 0.55}}}
end_power_model_update
```

---

## 9. Common UPF Mistakes (Interview Gold)

### Mistake 1: Missing Isolation on ALL Outputs

```
Problem: Designer isolates data_out but forgets about data_valid, interrupt,
         error_flag, and other control signals.

Consequence: When domain powers off, these control signals float to X,
             potentially triggering false interrupts or bus errors.

Fix: Use -applies_to outputs to catch ALL outputs. If some outputs need
     different clamp values, use multiple set_isolation commands with
     -elements to target specific signals.
```

### Mistake 2: Wrong Clamp Value

```
Problem: Active-low signal clamped to 0 (asserted) instead of 1 (deasserted).

Example: chip_select_n clamped to 0 -> external memory thinks it's selected
         when CPU is powered off -> bus contention.

Fix: Use -clamp_value 1 for active-low signals, 0 for active-high.
     Or use OR-type isolation for active-low signals.
```

### Mistake 3: Missing Level Shifter on Cross-Voltage Signals

```
Problem: CPU runs at 0.75V, IO runs at 1.8V. Data bus between them has
         no level shifter. 0.75V signal doesn't reach the logic-1 threshold
         of 1.8V logic.

Consequence: Functional failure -- the 1.8V receiver sees the 0.75V signal
             as indeterminate or logic-0.

Fix: Ensure set_level_shifter covers ALL cross-voltage signals.
     Use -rule both to catch signals in both directions.
```

### Mistake 4: Retention Restore Before Power Stable

```
Problem: Power-up sequence asserts RESTORE before VDD_CPU has fully ramped up.
         Shadow latch tries to transfer data at low voltage -> corrupted bits.

Consequence: CPU wakes up with wrong register values -> silent data corruption
             (worst kind of bug -- may not crash immediately but produce wrong results).

Fix: Insert sufficient delay or use a voltage detector to ensure supply is
     within 90-95% of nominal before asserting RESTORE.
```

### Mistake 5: Isolation Asserted Too Late

```
Problem: Power switch turns off BEFORE isolation is asserted.
         During the brief window: outputs are undefined AND not isolated.

Consequence: Glitches/X values propagate to always-on domain for a few ns.
             Can cause spurious writes, false interrupts, bus errors.

Fix: Assert isolation FIRST, wait for it to take effect (1-2 clock cycles),
     THEN turn off power switch. UPF specifies the intent; the control
     sequencing must be designed correctly in the power management FSM.
```

### Mistake 6: Forgetting Always-On Buffers

```
Problem: Clock signal enters a gated domain, gets buffered by standard cells
         (which are powered by the gated supply), then reaches retention FFs.

Consequence: When domain is off, clock buffer is dead, clock cannot reach
             retention FFs for save/restore operations.

Fix: Use always-on buffers for clock, save, restore, and isolation signals
     within a gated domain. UPF set_repeater or manual instantiation.
```

### Mistake 7: Feedback Paths Across Domains

```
Problem: Signal goes from PD_A to PD_B and back to PD_A. When PD_B is off:
         - PD_A -> PD_B: isolated (correctly clamped)
         - PD_B -> PD_A: isolated (correctly clamped)
         - But the clamped feedback value may not be what PD_A expects

Consequence: PD_A's logic receives a static clamped value instead of the
             dynamic feedback -> functional hang or incorrect behavior.

Fix: Carefully analyze all cross-domain feedback paths. May need latch
     isolation (to hold last valid value) instead of clamp isolation.
```

---

## 10. UPF for Multi-Voltage Design (DVFS Domains)

When domains operate at different voltages (not just on/off):

```tcl
# Domain running at variable voltage via DVFS
create_power_domain PD_CPU_DVFS \
    -elements {cpu_core}

# Multiple supply nets at different voltages
create_supply_net VDD_CPU_HIGH  -domain PD_CPU_DVFS  ;# 1.05V
create_supply_net VDD_CPU_NOM   -domain PD_CPU_DVFS  ;# 0.90V
create_supply_net VDD_CPU_LOW   -domain PD_CPU_DVFS  ;# 0.75V

# In practice, DVFS uses a single net whose voltage changes dynamically.
# UPF models this with power states:
add_power_state SS_CPU_DVFS \
    -state {TURBO   -supply_expr {power == `{FULL_ON, 1.05}}} \
    -state {NOMINAL -supply_expr {power == `{FULL_ON, 0.90}}} \
    -state {SVS     -supply_expr {power == `{FULL_ON, 0.75}}} \
    -state {OFF     -supply_expr {power == `{OFF}}}

# Level shifters needed at boundary to always-on domain (which stays at 0.9V)
set_level_shifter CPU_LS \
    -domain PD_CPU_DVFS \
    -applies_to both \
    -rule both \
    -name CPU_DVFS_LS
# "rule both" means: insert level shifter whether CPU voltage is higher OR lower
# than the receiving domain. At TURBO (1.05V), CPU is higher; at SVS (0.75V), lower.
```

---

## 11. UPF Simulation Testbench Patterns

### 11.1 Power Control Sequence in Testbench

```verilog
// Testbench power control using UPF supply control tasks
// These are simulator-specific tasks that control power state

// Synopsys VCS:
initial begin
    // Start with all domains ON
    $supply_on("soc_top/VDD", 0.9);
    $supply_on("soc_top/VDD_CPU", 0.9);
    $supply_on("soc_top/VDD_PERI", 0.9);
    
    // Run some functional cycles
    #1000;
    
    // Power down CPU domain
    $display("Powering down CPU...");
    cpu_iso_en = 1;           // Assert isolation
    #10;
    cpu_save = 1;             // Save retention state
    #5;
    cpu_save = 0;
    #5;
    $supply_off("soc_top/VDD_CPU");  // Turn off supply
    
    // Verify: all CPU outputs should be clamped (not X)
    #100;
    
    // Power up CPU domain
    $display("Powering up CPU...");
    $supply_on("soc_top/VDD_CPU", 0.9);  // Turn on supply
    #50;  // Wait for voltage stable
    cpu_restore = 1;          // Restore retention state
    #5;
    cpu_restore = 0;
    #5;
    cpu_iso_en = 0;           // Deassert isolation
    
    // CPU should now have its pre-power-down register values
end
```

### 11.2 Power-Aware Assertions

```systemverilog
// Assert that no X values reach always-on domain during power transitions
property no_x_on_aon_bus;
    @(posedge clk) disable iff (!reset_n)
    !$isunknown(aon_bus_data);
endproperty
assert property (no_x_on_aon_bus) else
    $error("X detected on always-on bus! Check isolation.");

// Assert correct power sequence
property iso_before_power_off;
    @(posedge clk)
    $fell(cpu_sleep_n) |-> cpu_iso_en;
    // When power switch goes off, isolation must already be asserted
endproperty
assert property (iso_before_power_off) else
    $error("Power off without isolation! Sequence error.");

// Assert retention restore only when power is stable
property restore_after_power;
    @(posedge clk)
    $rose(cpu_restore) |-> (cpu_sleep_n == 1'b0);
    // Restore only when power switch is ON (sleep_n is active-low here)
endproperty
```

---

## 12. Interview Questions and Answers

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

---

*This document targets senior-engineer / staff-level ASIC power interview preparation.
Cross-reference with Power_Fundamentals.md, Power_Reduction_Techniques.md, and
Power_Analysis_and_Signoff.md.*
