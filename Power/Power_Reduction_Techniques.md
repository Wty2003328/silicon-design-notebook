# Power Reduction Techniques for Digital IC / ASIC Design

## 1. Overview of Power Reduction Strategies

Power reduction techniques span all abstraction levels. The higher in the stack you apply
them, the greater the potential savings -- but also the greater the design effort.

| Level          | Techniques                                          | Typical Impact |
|----------------|-----------------------------------------------------|----------------|
| System/Arch    | DVFS, power gating, dark silicon, HW accelerators   | 10-100x        |
| Micro-arch     | Clock gating, operand isolation, memory partitioning | 2-10x          |
| Logic/RTL      | Encoding, resource sharing, FSM optimization         | 10-30%         |
| Gate-level     | Multi-Vt, gate sizing, buffer optimization           | 10-30%         |
| Circuit        | Adiabatic, body biasing, custom cells                | 5-20%          |
| Physical       | Wire optimization, placement for activity            | 5-15%          |

---

## 2. Clock Gating -- Deep Dive

Clock distribution typically consumes 30-50% of total dynamic power in a synchronous design.
Every flip-flop input sees a toggling clock every cycle, regardless of whether new data is
being captured. Clock gating eliminates unnecessary clock toggles to idle registers.

### 2.1 Basic Concept

```
Without clock gating:
  always @(posedge clk)
    q <= d;            // Clock toggles every cycle regardless

With clock gating:
  always @(posedge clk)
    if (enable)
      q <= d;          // Synthesizer infers ICG cell: gated_clk = clk & enable(latched)
```

### 2.2 ICG (Integrated Clock Gating) Cell -- Transistor-Level Design

There are three common ICG architectures:

**Architecture 1: Latch-AND (Industry Standard)**

```
              ___
  EN ------->|   |
             | L |---+
  CLK ------>|___|   |     _____
   (active   latch   +--->|     |
    low       (neg    |    | AND |---> GCLK
    transparent)  EN_L+--->|_____|
                           ^
  CLK --------------------|
```

The enable signal is latched on the NEGATIVE edge of the clock (latch is transparent
when CLK=0). The latched enable is ANDed with CLK to produce the gated clock.

**Why latch on negative edge?**
The enable must be stable during the positive edge of the clock (when flip-flops capture
data). Latching on the negative edge gives a full half-cycle setup window for the enable
signal. If enable changes during CLK=1 (positive phase), the latch is opaque and the
change is blocked until the next negative edge.

**Timing constraint on enable:**
```
  Enable must be stable by: t_falling_edge - t_setup_ICG

  Typical setup time of ICG cell: 50-200 ps (process dependent)
  This setup is to the FALLING clock edge, not the rising edge.
```

**Transistor-level implementation (Latch-AND):**
```
                  Vdd
                   |
                [PMOS]--- (CLK_b drives gate)
                   |
  EN ----+----[PMOS]--- (EN_b)     Transmission gate latch
         |         |                (transparent when CLK=0)
         |     [inv]---EN_latched
         |         |
  EN ----+----[NMOS]--- (EN)
                   |
                [NMOS]--- (CLK drives gate)
                   |
                  GND

  Then: GCLK = EN_latched AND CLK  (2-input AND or NAND+INV)
```

**Architecture 2: AND-based (no latch, NOT recommended)**

```
  GCLK = CLK AND EN

  Problem: glitches on EN during CLK=1 propagate directly to GCLK
  -> Can create clock glitches -> metastability / data corruption
  -> NEVER used in production designs for this reason
```

**Architecture 3: NAND-based with inverted clock convention**

```
  GCLK = NOT(NOT(CLK) NAND EN_latched)

  Functionally equivalent to Latch-AND but uses NAND which may be faster
  in some cell libraries. Output clock is active-high same as input.
```

### 2.3 Clock Gating Efficiency (CGE) and Power Savings Calculation

CGE = fraction of clock cycles where the clock is gated (disabled):

```
CGE = N_gated_cycles / N_total_cycles

Power savings from clock gating a group of N flip-flops:

P_saved = CGE * N * C_clk_per_FF * Vdd^2 * f_clk

where C_clk_per_FF includes:
  - Clock pin capacitance of the flip-flop (~2-5 fF)
  - Clock wire capacitance to reach that FF (~2-10 fF)
  - Clock buffer drive capacitance allocated per FF (~1-3 fF)
  Total: ~5-18 fF per FF (let's use 10 fF as typical)
```

**Numerical Example:**
```
Given:
  N = 1000 flip-flops in a register file
  C_clk_per_FF = 10 fF (including wire and buffer share)
  f_clk = 500 MHz
  Vdd = 0.9V
  CGE = 0.70 (register file accessed 30% of cycles)

P_saved = 0.70 * 1000 * 10e-15 * (0.9)^2 * 500e6
        = 0.70 * 10e-12 * 0.81 * 5e8
        = 0.70 * 4.05e-3
        = 2.835 mW

ICG cell overhead:
  Area: ~2x a minimum inverter = ~1 um^2 in 28nm
  Power: ~5 uW per ICG (clock input toggles, adds small load)
  Number of ICG cells: maybe 50-100 (one per 10-20 FFs)
  Total overhead: 50 * 5uW = 0.25 mW

Net savings: 2.835 - 0.25 = ~2.6 mW
```

### 2.4 Hierarchical Clock Gating

```
Level 1: Module-level gating
  - Gate the entire clock to a submodule when it's idle
  - Example: gate clock to UART when UART is disabled
  - CGE can be very high (>95% for rarely-used peripherals)
  - Controlled by software or power management unit

Level 2: Cluster-level gating
  - Gate groups of related registers (e.g., a pipeline stage)
  - 8-32 FFs per ICG cell
  - Medium CGE (40-70%)

Level 3: Fine-grain (RTL-inferred) gating
  - Synthesis tool inserts ICG for "if(en) q <= d" patterns
  - 1-8 FFs per ICG cell (configurable minimum)
  - CGE varies widely (10-90%)
```

Area overhead trade-off:
- Each ICG cell costs ~1-2x minimum inverter area
- Fine-grain gating: 5-8% area overhead typical
- But fine-grain gating saves more power than coarse-grain (captures more idle cycles)
- Optimal: combine hierarchical (coarse) + fine-grain, let synthesis handle the rest

### 2.5 Synthesis Tool Commands for Clock Gating

```tcl
# Synopsys Design Compiler
# Enable clock gating insertion
set_clock_gating_style \
  -sequential_cell latch \
  -positive_edge_logic {integrated} \
  -negative_edge_logic {integrated} \
  -control_point before \
  -control_signal scan_enable \
  -minimum_bitwidth 4 \
  -max_fanout 64

# Minimum bitwidth: don't insert ICG for fewer than 4 FFs (overhead not worth it)
# Max fanout: limit ICG fanout to 64 FFs (CTS will struggle with more)

# Report clock gating results
report_clock_gating -nosplit > clock_gating.rpt

# Cadence Genus
set_db design_clock_gating_max_fanout 32
set_db design_clock_gating_min_bitwidth 3
set_attribute lp_insert_clock_gating true [current_design]
```

### 2.6 Clock Gating and Scan (DFT Considerations)

During scan shift, ALL flip-flops must be clocked every cycle. Clock gating would block
the scan clock. Solution:

```
  GCLK = CLK AND (EN_latched OR SCAN_ENABLE)

  During scan: SCAN_ENABLE=1 -> GCLK = CLK always (gating bypassed)
  During functional mode: SCAN_ENABLE=0 -> GCLK = CLK AND EN_latched
```

The ICG cell has a dedicated test/scan_enable pin for this purpose.

---

## 3. Dynamic Voltage and Frequency Scaling (DVFS) -- Deep Dive

### 3.1 Voltage-Frequency Relationship from Gate Delay

Gate delay in short-channel regime (velocity saturation):

```
T_d = C_L * Vdd / I_drive

I_drive ~ mu * Cox * (W/L) * (Vgs - Vth)^alpha

where alpha = velocity saturation factor:
  alpha = 2.0 for long-channel (square law)
  alpha = 1.0-1.4 for short-channel (velocity saturated)
  alpha ~ 1.3 is typical for modern FinFET

Therefore:
  T_d ~ C_L * Vdd / ((Vdd - Vth)^alpha)
  f_max ~ (Vdd - Vth)^alpha / (C_L * Vdd)
```

For alpha = 1.3:
```
  f_max proportional to (Vdd - Vth)^1.3 / Vdd

  Example (Vth = 0.3V):
    Vdd = 1.0V: f ~ (0.7)^1.3 / 1.0 = 0.636
    Vdd = 0.9V: f ~ (0.6)^1.3 / 0.9 = 0.576  -> 9.4% slower
    Vdd = 0.8V: f ~ (0.5)^1.3 / 0.8 = 0.505  -> 20.5% slower
    Vdd = 0.7V: f ~ (0.4)^1.3 / 0.7 = 0.420  -> 34.0% slower
    Vdd = 0.5V: f ~ (0.2)^1.3 / 0.5 = 0.214  -> 66.4% slower
```

### 3.2 Power Savings Calculation

```
P_dynamic = C * Vdd^2 * f

If f scales as (Vdd - Vth)^alpha / Vdd:
  P ~ C * Vdd^2 * (Vdd - Vth)^alpha / Vdd
  P ~ C * Vdd * (Vdd - Vth)^alpha

For a 20% Vdd reduction (1.0V -> 0.8V, Vth=0.3V):
  f ratio = (0.5/0.8) / (0.7/1.0)^1.3 = (0.5^1.3/0.8) / (0.7^1.3/1.0)
          = 0.505 / 0.636 = 0.794  -> 20.6% slower
  P ratio = (0.8 * 0.5^1.3) / (1.0 * 0.7^1.3)
          = (0.8 * 0.4040) / (1.0 * 0.6360) = 0.323 / 0.636 = 0.508 -> 49.2% power savings!

Summary: 20% voltage reduction -> ~21% frequency loss -> ~49% dynamic power savings
This is why DVFS is the most powerful knob for dynamic power.
```

### 3.3 Operating Performance Points (OPPs)

A real-world DVFS table for a mobile SoC (representative values):

| OPP Name     | Voltage (V) | Frequency (GHz) | Dynamic Power | Leakage Power | Use Case              |
|--------------|-------------|------------------|---------------|---------------|-----------------------|
| Turbo        | 1.05        | 2.8              | 100% (ref)    | 100%          | Peak benchmark        |
| Nominal      | 0.90        | 2.0              | 55%           | 70%           | Sustained workload    |
| SVS          | 0.75        | 1.2              | 27%           | 45%           | Medium workload       |
| SVS-L1       | 0.65        | 0.8              | 15%           | 30%           | Light workload        |
| Low Power    | 0.55        | 0.4              | 6%            | 18%           | Background tasks      |
| Retention    | 0.50        | 0 (clock off)    | 0%            | 12%           | State retention only  |

### 3.4 DVFS Controller Architecture

```
                    +-------------------+
  Software DVFS     |  Performance      |
  Governor (Linux)  |  Monitoring Unit  |
  (cpufreq)         |  (PMU counters)   |
        |           +--------+----------+
        v                    |
  +-----+--------+          |
  | DVFS          |<---------+
  | Controller    |
  | (Hardware)    |
  +---+------+----+
      |      |
      v      v
  +---+--+ +-+------+
  | PMIC | | PLL /  |
  | (Vdd)| | Divider|
  +------+ +--------+
```

**Voltage-Frequency Change Sequence (CRITICAL for interviews):**

**Increasing performance (scaling UP):**
```
1. Increase voltage FIRST (wait for PMIC settling: 10-50 us typical)
2. THEN increase frequency (PLL relock: 5-20 us)

Why this order: if you increase frequency first at the old (lower) voltage,
timing violations occur because gates are too slow -> functional failure.
```

**Decreasing performance (scaling DOWN):**
```
1. Decrease frequency FIRST (divider change: <1 us, or PLL relock: 5-20 us)
2. THEN decrease voltage (wait for PMIC settling: 10-50 us)

Why this order: if you decrease voltage first at the old (higher) frequency,
same problem -- timing violations at the new lower voltage.
```

**Total transition time:** 20-100 us depending on PMIC speed and PLL architecture.
During this time, the processor is operational but at a non-optimal operating point.

### 3.5 Adaptive Voltage Scaling (AVS)

Silicon varies. Two chips from the same wafer can have 10-20% speed difference due to
process variation. DVFS uses a fixed voltage for each frequency -- sized for the WORST
chip. AVS adjusts voltage per-chip based on actual silicon speed.

**Implementation approaches:**

1. **Ring oscillator monitors:**
   ```
   Place ring oscillators on-die that mimic critical path delay.
   Ring oscillator frequency indicates actual silicon speed.
   
   Fast silicon -> ring osc runs fast -> reduce voltage -> save power
   Slow silicon -> ring osc runs slow -> increase voltage to meet timing
   
   Savings: 50-100mV on fast silicon -> 10-20% power reduction
   ```

2. **Critical path replica monitors:**
   ```
   Replicate actual critical paths and measure their delay
   More accurate than ring oscillators but harder to design/maintain
   Used in high-performance CPUs (Intel, AMD)
   ```

3. **In-situ timing monitors (speed sensors):**
   ```
   Place shadow latches near critical endpoints that detect timing violations
   If violation detected -> increase voltage
   If no violations for N cycles -> decrease voltage
   Most aggressive approach: operates at the edge of failure
   Risk: must detect and recover from actual timing errors
   Used in some ARM designs (adaptive clocking + voltage)
   ```

### 3.6 Per-Domain DVFS

Modern SoCs have independent voltage/frequency domains:

```
  +-----------+  +-----------+  +-----------+
  | CPU Core 0|  | CPU Core 1|  | GPU       |
  | Vdd_cpu0  |  | Vdd_cpu1  |  | Vdd_gpu   |
  | f_cpu0    |  | f_cpu1    |  | f_gpu     |
  +-----------+  +-----------+  +-----------+
  
  Each domain has:
  - Independent PMIC rail (or LDO)
  - Independent PLL/divider
  - Level shifters at domain boundaries
  - Handshake protocol for cross-domain communication
```

---

## 4. Power Gating -- Deep Dive

### 4.1 Concept

Power gating physically disconnects a logic block from its supply rails when it is idle,
reducing leakage to near zero (only the switch transistor leakage remains).

```
        Vdd (always on)
         |
      [PMOS header switch]  <-- controlled by SLEEP_N signal
         |
        VVDD (virtual Vdd, switched)
         |
    +----+----+
    | Logic   |
    | Block   |  <-- powered by virtual Vdd
    +----+----+
         |
        GND (or virtual GND with footer switch)
```

### 4.2 Header (PMOS) vs Footer (NMOS) Switches

**Header switch (PMOS between Vdd and VVDD):**
```
Advantages:
  - Logic sees clean ground (no ground bounce)
  - Most ASIC designs use this
  - PMOS in N-well can be isolated (no body effect issues with well taps)

Disadvantages:
  - PMOS is weaker than NMOS for same area (~2x larger needed)
  - More area overhead
```

**Footer switch (NMOS between VSS and VVSS):**
```
Advantages:
  - NMOS is stronger for same area (higher mobility)
  - Smaller switch area
  - Faster power-up (lower resistance for same size)

Disadvantages:
  - Virtual ground (VVSS) bounces during switching -> more noise
  - Ground bounce can corrupt data in retention registers
  - Can cause issues with ESD protection paths
```

**Industry practice:** Header switches are more common in ASIC designs. Footer switches
are sometimes used in SRAM arrays (where the regular structure helps manage ground bounce).

### 4.3 Switch Sizing

The power switch must be large enough to supply the active current of the gated block,
while also controlling inrush current during power-up.

**Steady-state sizing:**
```
I_active = P_active / Vdd  (active current of the logic block)

Switch on-resistance: R_sw = Vdd_drop / I_active

Vdd_drop target: typically 5-10% of Vdd (to avoid excessive performance degradation)

Example:
  P_active = 100 mW at Vdd = 0.9V
  I_active = 100mW / 0.9V = 111 mA
  Target IR drop = 5% * 0.9V = 45 mV
  R_sw = 45 mV / 111 mA = 0.405 ohms

  For a PMOS with R_on ~ 500 ohm*um (typical for 7nm header cell):
  Required total width = 500 / 0.405 = ~1235 um

  This is distributed across many switch cells (each ~5-10 um wide):
  Number of switch cells = 1235 / 7.5 = ~165 switch cells
```

**Inrush current (rush current) sizing:**
```
When the switch turns on, internal capacitance charges from 0 to Vdd.
If all switches turn on simultaneously:

I_rush = C_internal * dV/dt

where:
  C_internal = total decoupling + gate capacitance in the gated domain
  dV/dt = rate of voltage rise on virtual Vdd

To limit Ldi/dt noise on the package:
  I_rush < I_rush_max (set by package inductance and voltage noise budget)

Example:
  C_internal = 10 nF (typical for a moderate-size block)
  I_rush_max = 100 mA (from supply noise analysis)
  Required dV/dt = 100mA / 10nF = 10 MV/s = 10V/us
  Time to charge to 0.9V: 0.9V / (10V/us) = 90 ns

  If this is too slow, either:
  - Accept higher rush current (strengthen package decap)
  - Or use daisy-chain (staged) power-up
```

### 4.4 Daisy-Chain (Staged) Power-Up

Instead of turning on all switches simultaneously, enable them sequentially with a
staggered delay:

```
SLEEP_N ----[D]---> SW_EN_1 ----[D]---> SW_EN_2 ----[D]---> SW_EN_3 ...
             ^                    ^                    ^
             |                    |                    |
          delay cell          delay cell          delay cell

Waveforms:
          t0    t1    t2    t3    t4
SW_EN_1:  ____/```````````````````````
SW_EN_2:  __________/```````````````````
SW_EN_3:  ________________/```````````````
SW_EN_4:  ______________________/```````````
VVDD:     ____..../'''''''''''''/````````````
              gradual ramp     full Vdd
```

Each switch cell turns on after the previous one has been on for one delay step.
This limits the instantaneous current to roughly I_rush / N_stages.

**Implementation in UPF:**
```
create_power_switch CPU_SW \
  -domain PD_CPU \
  -input_supply_port {vin  VDD} \
  -output_supply_port {vout VDD_CPU} \
  -control_port {sleep_n CPU_SLEEP_N} \
  -on_state {on_state vin {!sleep_n}} \
  -off_state {off_state {sleep_n}}

# Physical implementation handles daisy-chain automatically
# during power switch insertion in ICC2/Innovus
```

### 4.5 Isolation Cells

When a power-gated domain is off, its outputs are undefined (floating/X). This can
corrupt always-on logic that receives these signals. Isolation cells clamp the outputs
to known values.

**Types of isolation cells:**

| Type           | Clamp Value | Implementation          | When to Use                      |
|----------------|-------------|-------------------------|----------------------------------|
| AND isolation  | 0           | AND(signal, !isolate)   | Data buses (force 0)             |
| OR isolation   | 1           | OR(signal, isolate)     | Active-low control signals       |
| Latch isolation| Last value  | Latch + MUX             | Status registers, config outputs |
| Clamp-low      | 0           | Tie to VSS when isolated| General data outputs             |
| Clamp-high     | 1           | Tie to VDD when isolated| Active-low resets, enables       |

**Isolation cell power supply:**
The isolation cell MUST be powered by the always-on supply (since it must function
when the gated domain is off). It is placed at the boundary of the power domain.

```
  +---[Gated Domain (OFF)]---+     +---[Always-On Domain]---+
  |                          |     |                        |
  |  signal -----> ISO_CELL -----> receiving_logic          |
  |                powered   |     |                        |
  |                by AO     |     |                        |
  +--------------------------+     +------------------------+
```

**Isolation control signal timing:**
```
Isolation MUST be asserted BEFORE power is removed:
  1. Assert isolation -> outputs clamped to safe values
  2. Turn off power switch -> domain goes dark
  
Isolation MUST be deasserted AFTER power is restored:
  1. Turn on power switch -> domain powers up
  2. Wait for voltage to stabilize
  3. Initialize/restore state
  4. Deassert isolation -> live outputs propagate
```

### 4.6 Retention Registers

When a power-gated domain is turned off, all flip-flop state is lost. For domains that
need fast wake-up (avoid full re-initialization), retention registers save state before
power-down and restore it after power-up.

**Balloon Latch Design:**
```
                        Always-On Supply (VAO)
                              |
                +-------------+-------------+
                |     Shadow Latch          |
  D --->[Master]--->[Slave]--->[Save/Restore]---> Q
         ^         ^    |     ^
         |         |    |     |
        CLK      CLK   +---->+ SAVE/RESTORE signals
                        |
              Switchable Supply (VVDD)
              
  Normal operation:
    Master-slave FF operates normally (powered by VVDD)
    Shadow latch is idle (powered by VAO)
  
  Save (before power-down):
    SAVE pulse copies slave output to shadow latch
    Shadow latch retains value (powered by always-on VAO)
  
  Power-down:
    VVDD goes to 0. Master and slave lose state.
    Shadow latch maintains state on VAO.
  
  Power-up:
    VVDD restored. Master and slave are in unknown state.
  
  Restore (after power-up, before functional operation):
    RESTORE pulse copies shadow latch value back to slave
    FF now has its pre-power-down value
```

**Area and power overhead of retention FFs:**
```
  Standard flip-flop:  ~16-20 transistors, area = 1x
  Retention flip-flop: ~28-36 transistors, area = 1.6-2.0x
  
  Always-on leakage of retention FF: ~2-5 nA per FF (for the shadow latch)
  For 10K retention FFs: 10,000 * 5nA = 50 uA * 0.9V = 45 uW always-on leakage
```

**Selective retention:**
```
Not all state needs retention. To minimize area/leakage:
  - Retain: CPU architectural registers, interrupt state, power management state,
    bus configuration, security context
  - Do NOT retain: cache data (re-fetch from memory), pipeline state (flush and
    restart), debug registers (reinitialize)
    
Selective retention can reduce retention FF count by 80-90%, saving significant area.
```

### 4.7 Complete Power-Up/Down Sequence

**Power-down sequence:**
```
  t0: Software requests power-down for domain PD_CPU
  t1: Hardware waits for pending transactions to complete (drain pipeline)
  t2: Disable clocks to the domain (clock gating)
  t3: SAVE pulse -> retention FFs capture state to shadow latches
  t4: Assert isolation -> outputs clamped to safe values
  t5: Assert sleep (turn off power switches) -> VVDD ramps down
  t6: Domain is fully off. Only always-on logic and retention latches draw power.
```

**Power-up sequence:**
```
  t0: Wake-up event (interrupt, timer, external signal)
  t1: Deassert sleep (turn on power switches, staged/daisy-chain)
      -> VVDD ramps up gradually
  t2: Wait for VVDD to reach stable voltage (monitor with voltage detector or
      fixed timer, typically 5-50 us)
  t3: Assert reset to the domain (put logic in known state)
  t4: RESTORE pulse -> retention FFs recover state from shadow latches
  t5: Deassert reset
  t6: Deassert isolation -> live outputs start propagating
  t7: Enable clocks
  t8: Domain is fully operational. Resume execution.
```

**Timing diagram:**
```
              power-down                    power-up
           t0 t1 t2 t3 t4 t5           t0 t1    t2  t3 t4 t5 t6 t7
           |  |  |  |  |  |            |  |      |   |  |  |  |  |
CLK_EN:    ```````\____________________|  |      |   |  |  |  |__/````
SAVE:      ________/`\________________|  |      |   |  |  |  |  |
ISO:       ___________/````````````````|``|``````|```|``|``|``\__|____
SLEEP_N:   ```````````````\____________|__/``````|   |  |  |  |  |
VVDD:      ``````````````````\_________|__..../``````|   |  |  |  |
RESET:     ___________________________________|``|```\__|  |  |
RESTORE:   _________________________________________/`\____|  |
OUTPUT:    ````````````XXXX|000000000000|000000000000XXX|``````````
                   valid   clamped      clamped   valid
```

### 4.8 State Retention Power Gating (SRPG) vs Full Power Gating

| Feature              | SRPG (with retention)        | Full Power Gating            |
|----------------------|------------------------------|------------------------------|
| State preservation   | Yes (selected registers)     | No (all state lost)          |
| Area overhead        | ~10-20% (retention FFs)      | ~3-5% (switches + ISO only)  |
| Always-on leakage    | Higher (shadow latches)      | Lower (only ISO cells)       |
| Wake-up latency      | Fast (5-50 us)               | Slow (ms: full boot/init)    |
| Design complexity    | Higher                       | Lower                        |
| Use case             | CPU cores, DSPs              | Peripherals, rarely-used IPs |

---

## 5. Multi-Vt Optimization -- Deep Dive

### 5.1 Threshold Voltage from MOSFET Physics

```
Vth = Vth0 + gamma * (sqrt(2*phi_f + Vsb) - sqrt(2*phi_f))

where:
  Vth0 = zero-bias threshold voltage (determined by channel doping, ion implant dose)
  gamma = body effect coefficient = sqrt(2*q*epsilon_si*N_A) / Cox
  phi_f = Fermi potential = V_T * ln(N_A / n_i)
  Vsb = source-to-body voltage
```

Foundries create multiple Vth flavors by adjusting the channel implant dose:
- **Higher doping** -> higher Vth -> slower but much less leakage (HVT)
- **Lower doping** -> lower Vth -> faster but much more leakage (LVT)

In FinFET processes, Vth is controlled by work function metal (WFM) type/thickness
rather than channel doping (undoped channel for better mobility and variation).

### 5.2 Library Comparison Table (Typical 7nm FinFET)

| Vt Flavor | Vth (mV) | Delay (normalized) | Leakage (normalized) | Use Case                        |
|-----------|----------|---------------------|-----------------------|---------------------------------|
| UHVT      | ~400     | 1.50x               | 0.05x                 | Ultra-low-power, standby paths  |
| HVT       | ~350     | 1.20x               | 0.20x                 | Non-critical paths (majority)   |
| SVT       | ~300     | 1.00x (baseline)    | 1.00x                 | Moderate paths                  |
| LVT       | ~250     | 0.80x               | 5.0x                  | Near-critical paths             |
| ULVT      | ~200     | 0.65x               | 20.0x                 | Most critical paths only        |
| ELVT      | ~150     | 0.55x               | 80.0x                 | Speed emergency (sparingly)     |

**Key relationships:**
```
Every ~50mV reduction in Vth:
  - Speed improves by ~15-25% (depends on Vdd, process)
  - Leakage increases by ~4-5x (exponential: exp(50mV / (1.1*26mV)) = exp(1.75) = 5.75x)
```

### 5.3 Vt Optimization Algorithm

The standard approach is constraint-driven Vt assignment:

```
Algorithm: Multi-Vt Optimization

1. Start with ALL cells at HVT (minimum leakage)
2. Run timing analysis (STA)
3. Identify all timing-violating paths
4. For each violating path (worst first):
   a. Identify cells with most negative slack contribution
   b. Swap these cells from HVT -> SVT (or SVT -> LVT)
   c. Re-run incremental timing
   d. If timing met, stop swapping on this path
   e. If still violating, continue swapping more cells
5. Repeat until all paths meet timing
6. Post-optimization: check leakage budget
   - If over budget, consider trading: accept tighter timing margins,
     re-optimize with more aggressive clock uncertainty
7. Final: check timing at all corners (including leakage corner)
```

**Tool commands:**
```tcl
# Synopsys Design Compiler
set_multi_vth_constraint -lvth_percentage 15 -type soft
# Limit LVT cells to 15% of total cell count

# After optimization, check distribution:
report_threshold_voltage_group

# Cadence Innovus (post-route)
setMultiCpuUsage -localCpu 16
optDesign -postRoute -setup -hold
# Automatically performs Vt swapping as part of optimization

# Explicit Vt swap command:
swapCells -cell U123 -toCell NAND2_SVT
```

**Typical Vt distribution in a well-optimized design:**
```
  HVT:  55-65% (non-critical paths, vast majority)
  SVT:  20-30% (moderately timing-critical)
  LVT:  8-15% (near-critical paths)
  ULVT: 1-3%  (most critical paths only)
```

### 5.4 Leakage Savings Calculation

```
Example: 10M gate design, all SVT baseline

All-SVT leakage: 10M * 10 nA * 0.9V = 90 mW (at 25C)

After Vt optimization (60% HVT, 25% SVT, 12% LVT, 3% ULVT):
  HVT contribution:  6.0M * 10 * 0.20 * 0.9V = 10.8 mW
  SVT contribution:  2.5M * 10 * 1.00 * 0.9V = 22.5 mW
  LVT contribution:  1.2M * 10 * 5.00 * 0.9V = 54.0 mW
  ULVT contribution: 0.3M * 10 * 20.0 * 0.9V = 54.0 mW
  Total: ~141 mW

Wait, that's higher! The issue is that a small number of LVT/ULVT cells dominate leakage.

This illustrates the key insight: even 3% ULVT cells contribute 38% of total leakage.
The optimization must minimize the use of fast Vt cells to only truly critical paths.

Better optimization (60% HVT, 30% SVT, 9% LVT, 1% ULVT):
  HVT:  6.0M * 10nA * 0.20 * 0.9V = 10.8 mW
  SVT:  3.0M * 10nA * 1.00 * 0.9V = 27.0 mW
  LVT:  0.9M * 10nA * 5.00 * 0.9V = 40.5 mW
  ULVT: 0.1M * 10nA * 20.0 * 0.9V = 18.0 mW
  Total: ~96 mW -- only slightly more than all-SVT

vs all-HVT: 10M * 10nA * 0.20 * 0.9V = 18 mW (but won't meet timing)

The art is in minimizing the LVT/ULVT count while still meeting timing.
```

---

## 6. Body Biasing

### 6.1 Forward Body Bias (FBB)

Apply a small forward bias to the body-source junction:
```
For NMOS: Vbs > 0 (body slightly positive relative to source)
  -> Vth decreases (body effect equation with negative Vsb)
  -> Faster switching (more drive current)
  -> More leakage (lower Vth -> exponentially more subthreshold current)

Typical FBB: 100-300 mV forward
  -> Vth reduction: 30-80 mV
  -> Speed improvement: 10-25%
  -> Leakage increase: 3-10x
```

### 6.2 Reverse Body Bias (RBB)

Apply a reverse bias to increase the body-source junction voltage:
```
For NMOS: Vbs < 0 (body negative relative to source, or equivalently Vsb > 0)
  -> Vth increases (body effect equation with positive Vsb)
  -> Slower switching
  -> Reduced leakage

Typical RBB: 100-500 mV reverse
  -> Vth increase: 30-120 mV
  -> Leakage reduction: 3-30x
  -> Speed degradation: 10-30%
```

### 6.3 Adaptive Body Biasing (ABB)

Combine FBB and RBB adaptively based on silicon speed and operating conditions:

```
  Fast silicon + high temperature: apply RBB to reduce leakage
  Slow silicon + low temperature: apply FBB to meet timing
  
  On-chip ring oscillator detects actual silicon speed
  ABB controller adjusts body bias voltage via on-chip charge pump or external regulator
```

### 6.4 Triple-Well Requirement

Standard bulk CMOS has the P-substrate common to all NMOS devices -- you cannot independently
bias NMOS bodies. Triple-well process adds a deep N-well under the P-well of NMOS devices,
isolating them:

```
  Standard bulk:     P-sub is shared -> no individual NMOS body bias
  Triple-well:       Deep N-well isolates P-well -> independent NMOS body bias per domain

  Cross-section:
  
  PMOS (in N-well)     NMOS (in isolated P-well)
  |  N+  P+ |          |  P+  N+ |
  |  S   B  |          |  B   S  |
  +----N-well----+     +----P-well----+
  |              |     |    Deep N-well     |
  +--------------+     +-------------------+
  |                    P-substrate          |
  +-----------------------------------------+
  
  Separate body bias can be applied to the isolated P-well
```

### 6.5 Body Bias Effectiveness at Advanced Nodes

Body biasing is LESS effective in FinFET processes because:
- The channel is a thin fin with very little body to bias
- Gate control dominates over body effect
- Available body bias range is limited

At 7nm and below, body biasing provides only 5-15% Vth modulation (vs 20-40% in planar).
This is why multi-Vt libraries (WFM-based) are the primary lever in FinFET designs,
and body biasing is a supplementary technique.

---

## 7. Operand Isolation

### 7.1 Concept

When a combinational block's output is not needed (e.g., the result will be discarded
because a MUX selects a different input), the inputs still toggle and waste power.
Operand isolation gates the inputs to prevent unnecessary switching.

### 7.2 RTL Pattern

```verilog
// WITHOUT operand isolation (wasteful):
wire [31:0] mult_result = a * b;  // Multiplier toggles every cycle
wire [31:0] add_result  = c + d;
assign result = sel ? mult_result : add_result;

// WITH operand isolation (power-efficient):
wire [31:0] a_gated = sel ? a : 32'b0;  // Freeze multiplier inputs when not selected
wire [31:0] b_gated = sel ? b : 32'b0;
wire [31:0] mult_result = a_gated * b_gated;  // No toggling when sel=0
wire [31:0] add_result  = c + d;
assign result = sel ? mult_result : add_result;
```

### 7.3 Synthesis Pragma

Many synthesis tools can insert operand isolation automatically:

```verilog
// Synopsys DC pragma
(* isolate_operand = "true" *)
wire [31:0] mult_result = a * b;

// Or via TCL constraint:
// set_operand_isolation -design my_block -elements {mult_instance}
```

### 7.4 Power Savings

For a 32-bit multiplier with random inputs:
```
Without isolation: multiplier toggles every cycle
  - 32x32 multiplier has ~3000-5000 internal gates
  - Average activity ~0.2 -> significant power

With isolation (sel active 30% of cycles):
  - Multiplier active only 30% of cycles
  - Power savings: ~70% of multiplier dynamic power
  
For a multiplier consuming 5 mW:
  Savings = 0.70 * 5 mW = 3.5 mW
```

**When to use:**
- Multipliers, dividers (high power, not always needed)
- Functional units behind MUXes (only one selected at a time)
- Bus interface blocks (active only during transactions)
- Any large combinational block with a valid/enable signal

---

## 8. Additional Power Reduction Techniques

### 8.1 Memory Power Optimization

Memories (SRAM, register files) often consume 40-60% of total chip power:

```
Memory power reduction techniques:
  1. Memory banking: divide large memory into banks, enable only the accessed bank
     - 8-bank memory: 7 banks idle -> ~87% dynamic power savings on inactive banks
  
  2. Peripheral clock gating: gate clocks to memory read/write circuits when idle
  
  3. Bit-line power: use hierarchical bit-lines (local + global) to reduce
     bit-line capacitance
  
  4. Memory shutdown: completely power-gate unused memory banks
     - Requires data migration or re-initialization
     - State-retentive SRAM available (retention mode at ~0.5V)
  
  5. Read-assist / Write-assist: allow lower Vdd operation for SRAM
     - Negative bit-line write: improves write margin at low Vdd
     - Word-line underdrive: reduces SRAM read disturbance at low Vdd
```

### 8.2 Data Encoding for Low Power

```
Bus Invert Coding:
  Compare new data word with previous word
  If Hamming distance > N/2 (more than half the bits change):
    Invert the data and send an invert flag
  
  Reduces worst-case bus transitions from N to N/2
  Average bus power reduction: ~20-30% for random data on wide buses
  Overhead: 1 extra wire (invert flag) + XOR logic at both ends
  
Gray Coding for counters/addresses:
  Only 1 bit changes per increment (vs up to N bits for binary)
  Counter power reduction: significant for wide counters
  Used in FIFO pointers, address generators
```

### 8.3 Voltage Islands

Different blocks can run at different voltages (but same or different frequencies):

```
  +---[CPU: 0.9V]---+   +---[GPU: 0.8V]---+   +---[IO: 1.8V]---+
  |                  |   |                  |   |                 |
  | Level shifters   |<->| Level shifters   |<->| Level shifters  |
  | at boundaries    |   | at boundaries    |   | at boundaries   |
  +------------------+   +------------------+   +-----------------+
  
Level shifters required at EVERY signal crossing between voltage domains.
  - Low-to-high: simple pull-up level shifter
  - High-to-low: more complex (need to handle full swing at destination)
  - Each level shifter adds ~1-3 gate delays of latency
```

### 8.4 Adiabatic and Energy Recovery Logic

Conceptually interesting but rarely practical in digital ASIC:
```
  Conventional CMOS: energy dissipated = C*Vdd^2 per cycle (fundamentally limited)
  Adiabatic: use a ramped (AC) power supply that charges/discharges C slowly
    - Energy dissipated = (R*C/T) * C*Vdd^2, where T = ramp time
    - If T >> RC, dissipation approaches zero
    - Requires multi-phase clock generation and distribution
    - Performance is low (slow ramp times)
    - Area overhead is high
  Used in: some ultra-low-power sensor nodes, RFID tags
```

---

## 9. Interview Questions and Answers

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
```
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

---

## 10. Summary: Power Reduction Decision Tree

```
Is the block idle for extended periods (ms)?
  YES -> Power gating (eliminate leakage)
         Need fast wake-up? -> SRPG with retention
         Can tolerate slow wake-up? -> Full power gating (simpler)
  
  NO -> Does workload intensity vary?
    YES -> DVFS (scale voltage/frequency with demand)
           Is demand unpredictable? -> Add AVS for per-chip optimization
    
    NO -> Is clock power dominant?
      YES -> Clock gating (always do this regardless)
             Check CGE metric, target >60% for each block
      
      NO -> Is leakage dominant?
        YES -> Multi-Vt optimization (target <10% LVT/ULVT)
               Consider reverse body bias for standby mode
        
        NO -> Dynamic power optimization
              Operand isolation (gate unused functional units)
              Data encoding (bus invert for wide buses)
              Memory banking (activate only needed banks)
              Path balancing (reduce glitch power)
```

---

*This document targets senior-engineer / staff-level ASIC power interview preparation.
Cross-reference with Power_Fundamentals.md, UPF_Power_Intent.md, and
Power_Analysis_and_Signoff.md.*
