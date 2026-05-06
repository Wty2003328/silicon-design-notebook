# Static Timing Analysis (STA) -- The Complete Interview Bible

## Table of Contents
1. Cell Delay Modeling (NLDM, CCS)
2. Timing Arcs and Polarity
3. Setup and Hold -- Transistor-Level Derivation
4. Slack Calculation -- PrimeTime Report Walkthrough
5. Multi-Cycle Paths with Waveform Proofs
6. Clock Reconvergence Pessimism Removal (CRPR/CPPR)
7. OCV / AOCV / POCV Deep Dive
8. Crosstalk and Signal Integrity in STA
9. Timing ECO Flow
10. Full SDC Walkthrough for a Realistic SoC
11. Clock Tree Synthesis (CTS)
12. MCMM and Signoff Methodology
13. Interview Q&A (25+ Questions)

---

## 1. Cell Delay Modeling

### 1.1 NLDM (Non-Linear Delay Model)

NLDM is the traditional .lib delay model used from 180nm through 28nm. Each timing arc has a 2D lookup table indexed by:

- **Input transition time** (input slew, rise or fall)
- **Output load capacitance** (total capacitance seen at output pin)

The table stores **cell delay** and **output slew** values.

```
/* Example NLDM table for a NAND2 cell, rise delay */
cell_rise (delay_template_7x7) {
  index_1 ("0.01, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0");    /* input slew (ns) */
  index_2 ("0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5");  /* output load (pF) */
  values ( \
    "0.022, 0.028, 0.039, 0.067, 0.112, 0.201, 0.469", \
    "0.031, 0.037, 0.048, 0.076, 0.121, 0.210, 0.478", \
    "0.041, 0.047, 0.058, 0.086, 0.131, 0.220, 0.488", \
    "0.058, 0.064, 0.075, 0.103, 0.148, 0.237, 0.505", \
    "0.108, 0.114, 0.125, 0.153, 0.198, 0.287, 0.555", \
    "0.195, 0.201, 0.212, 0.240, 0.285, 0.374, 0.642", \
    "0.368, 0.374, 0.385, 0.413, 0.458, 0.547, 0.815"  \
  );
}
```

**How the tool uses NLDM:**

1. The tool knows the input slew (from the driving cell's output slew) and the output load (from fanout pin capacitances + wire capacitance extracted from SPEF).
2. It performs bilinear interpolation in the table to get cell delay and output slew.
3. The output slew becomes the input slew for the next stage.

**Numerical example:**

```
Given: Input slew = 0.15 ns, Output load = 0.03 pF
Table indices:
  Slew:  between index 0.1 and 0.2
  Load:  between index 0.02 and 0.05

Bilinear interpolation:
  At slew=0.1, load=0.02: 0.058
  At slew=0.1, load=0.05: 0.086
  At slew=0.2, load=0.02: 0.075
  At slew=0.2, load=0.05: 0.103

  Interpolated at slew=0.15:
    load=0.02: (0.058 + 0.075)/2 = 0.0665
    load=0.05: (0.086 + 0.103)/2 = 0.0945

  Interpolated at load=0.03:
    fraction = (0.03 - 0.02)/(0.05 - 0.02) = 0.333
    delay = 0.0665 + 0.333 * (0.0945 - 0.0665) = 0.0758 ns
```

**NLDM limitations:**

- Models the output as a **single lumped capacitance** -- ignores the distributed RC nature of real interconnect.
- At advanced nodes (16nm and below), the output waveform shape matters because the effective load is not purely capacitive (significant resistive shielding from interconnect).
- NLDM uses a single number for delay and slew -- does not capture waveform distortion.

### 1.2 CCS (Composite Current Source) Model

CCS (Synopsys) and ECSM (Cadence's Effective Current Source Model) address NLDM's limitations by modeling the cell as a **time-varying current source** driving into an RC load.

**Why CCS is needed at 7nm and below:**

1. **Waveform shape matters**: At advanced nodes, the voltage waveform at the output is not a clean ramp. It has a "bump" due to Miller capacitance and different stages of the output driver turning on. This distorted waveform propagates differently through downstream gates.

2. **Effective capacitance**: The interconnect is a distributed RC network. The near-end capacitance charges quickly, but the far-end capacitance charges slowly through interconnect resistance. NLDM sees only a lumped C, so it over-estimates delay for fast transitions and under-estimates for slow transitions.

3. **Receiver-pin dependent delay**: The delay to different fanout pins varies depending on their position in the RC network. CCS can model this; NLDM cannot.

**CCS data in .lib:**

```
/* CCS models store current waveforms instead of delay values */
output_current_rise (ccs_template) {
  index_1 ("0.01, 0.05, 0.1, 0.2");          /* input slew */
  index_2 ("0.005, 0.01, 0.02, 0.05");        /* output load */
  index_3 ("0.0, 0.1, 0.2, 0.3, 0.5, 1.0");  /* time points (ns) */
  values ( \
    /* Current values at each time point for each slew/load combo */
    ...
  );
}

/* Also stores receiver capacitance models */
receiver_capacitance1_rise (ccs_template) { ... }
receiver_capacitance2_rise (ccs_template) { ... }
```

The tool performs a SPICE-like simulation using the current source model and the extracted RC interconnect to compute:
- Accurate delay to each receiver pin
- Accurate slew at each receiver pin
- Waveform shape for downstream propagation

**CCS vs NLDM accuracy comparison:**

| Metric              | NLDM          | CCS/ECSM      |
|---------------------|---------------|----------------|
| Delay accuracy      | +/- 10-15%    | +/- 2-3%       |
| Slew accuracy       | +/- 15-20%    | +/- 3-5%       |
| Waveform modeling   | None          | Full waveform  |
| Library size        | ~50 MB        | ~200-500 MB    |
| Runtime overhead    | Baseline      | 2-4x slower    |
| Recommended nodes   | >= 28nm       | <= 16nm        |

### 1.3 Wire Delay Models

**Elmore delay** (simple estimate):

```
For a single RC segment: T_Elmore = R * C
For distributed RC:      T_Elmore = sum(R_i * C_downstream_i)
```

**Arnoldi/AWE (Asymptotic Waveform Evaluation):**

More accurate than Elmore for multi-segment RC networks. Used in signoff tools with SPEF extraction.

**SPEF (Standard Parasitic Exchange Format):**

```
*D_NET total_cap_net 0.0523
*CONN
*I inst1/Z O
*I inst2/A I
*I inst3/A I
*CAP
1 inst1/Z 0.0102
2 inst2/A 0.0087
3 n1 0.0156
4 inst3/A 0.0078
5 n2 0.0100
*RES
1 inst1/Z n1 2.45
2 n1 inst2/A 1.83
3 n1 n2 3.12
4 n2 inst3/A 2.67
*END
```

---

## 2. Timing Arcs and Polarity

### 2.1 Types of Timing Arcs

**Combinational arcs:** Delay from input pin to output pin of a combinational cell.

```
cell (AND2) {
  pin (A) { direction: input; }
  pin (B) { direction: input; }
  pin (Y) { direction: output;
    timing () {
      related_pin: "A";
      timing_type: combinational;
      cell_rise (...) { ... }
      cell_fall (...) { ... }
      rise_transition (...) { ... }
      fall_transition (...) { ... }
    }
    timing () {
      related_pin: "B";
      timing_type: combinational;
      ...
    }
  }
}
```

**Sequential arcs:**

| Arc Type        | Definition                                                  | Timing Type in .lib         |
|-----------------|-------------------------------------------------------------|-----------------------------|
| Setup           | Min time data must be stable BEFORE clock edge              | setup_rising / setup_falling|
| Hold            | Min time data must be stable AFTER clock edge               | hold_rising / hold_falling  |
| Clock-to-Q      | Delay from clock edge to Q output                          | rising_edge / falling_edge  |
| Recovery        | Min time async signal must be stable BEFORE clock edge      | recovery_rising             |
| Removal         | Min time async signal must be stable AFTER clock edge       | removal_rising              |
| Min pulse width | Minimum clock pulse width (high or low)                     | min_pulse_width_high/low    |

### 2.2 Timing Arc Polarity (Unateness)

**Positive unate:** Output transitions in the SAME direction as input.

```
Buffer:  A rises -> Y rises    (positive unate)
         A falls -> Y falls

AND gate (from A, with B=1):
         A rises -> Y rises    (positive unate on A)
         A falls -> Y falls
```

**Negative unate:** Output transitions in the OPPOSITE direction.

```
Inverter:  A rises -> Y falls  (negative unate)
           A falls -> Y rises

NAND gate (from A, with B=1):
           A rises -> Y falls  (negative unate on A)
           A falls -> Y rises
```

**Non-unate:** Output direction depends on OTHER input values.

```
XOR gate:
  When B=0: A rises -> Y rises  (positive)
  When B=1: A rises -> Y falls  (negative)

  Since the polarity depends on B, the A->Y arc is NON-UNATE.
```

**Why unateness matters:**

- For **positive unate** arcs, rise delay is associated with rise input, fall with fall input.
- For **negative unate** arcs, rise delay is associated with fall input, fall with rise input.
- For **non-unate** arcs, the tool must check BOTH rise and fall combinations (worst-case), doubling the analysis work.

In .lib:

```
timing () {
  related_pin: "A";
  timing_sense: positive_unate;  /* or negative_unate, non_unate */
}
```

### 2.3 Timing Arc Conditions

Conditional arcs model data-dependent delays (e.g., different delays through a MUX depending on select):

```
timing () {
  related_pin: "A";
  when: "S";           /* This arc is active only when S=1 */
  sdf_cond: "S==1";
  timing_sense: positive_unate;
  cell_rise (...) { ... }
}
```

Used in SDF back-annotation for maximum accuracy.

---

## 3. Setup and Hold -- Transistor-Level Derivation

### 3.1 Inside a Master-Slave Edge-Triggered Flip-Flop

A standard positive-edge-triggered D-FF consists of two transmission-gate latches:

```
                CLK=0 pass    CLK=1 pass
  D ---[TG1]---o M_node ---[TG2]---o S_node --- Q
         |                    |
      [INV1]               [INV2]
         |                    |
         +---feedback---+    +---feedback---+
              (CLKb)              (CLK)

  Master latch: transparent when CLK=0 (samples D)
  Slave latch:  transparent when CLK=1 (drives Q)
```

**On the rising clock edge (CLK: 0->1):**

1. TG1 closes (master latches data at internal node M)
2. TG2 opens (slave becomes transparent, M propagates to Q)
3. Q settles after Tc2q delay

### 3.2 Setup Time -- Physical Meaning

**Setup time** is the minimum time D must be stable BEFORE the clock edge so that the internal M_node reaches a voltage level sufficient for correct latching.

**What happens physically when setup is violated:**

1. D changes too close to the clock edge.
2. TG1 closes while M_node is still transitioning.
3. M_node is "caught" at an intermediate voltage (between VDD and GND).
4. The inverter feedback pair (INV1) has both PMOS and NMOS partially on.
5. M_node slowly resolves via the regenerative latch -- this is **metastability**.
6. If M resolves correctly before TG2 propagates it, we get a delayed but valid Q (increased Tc2q).
7. If M does not resolve in time, Q goes metastable.

The setup time is defined as the point where Tc2q degrades by X% (typically 10%) from its nominal value.

```
Tc2q
  |
  |                  *
  |                 *
  |               **
  |             **
  |   *********
  |---+--------+----+---> D arrival time before CLK edge
      |        |
      Tsu     "cliff" where Tc2q goes to infinity (metastability)
```

### 3.3 Hold Time -- Physical Meaning

**Hold time** is the minimum time D must remain stable AFTER the clock edge so that TG1 fully disconnects the input from M_node before D changes.

Physically, TG1 does not close instantaneously -- it takes a finite time for the clock to propagate through the local clock inverter and fully turn off the transmission gate. If D changes during this window, the new D value "leaks" into M_node and corrupts the latched data.

### 3.4 Negative Hold Time

Modern flip-flops in standard cell libraries often have negative hold time (e.g., -20ps). This means D can change BEFORE the clock edge and still be correctly captured. This occurs when:

1. The internal clock path to TG1 is slower than the external data path to the TG1 input.
2. The clock must propagate through internal inverters before it reaches TG1's control.
3. D can change up to |Th| before the external clock edge because TG1 hasn't received the clock transition yet.

Negative hold time is extremely desirable -- it makes hold timing easier to close.

### 3.5 Setup/Hold Analysis -- Full Numerical Walkthrough

**Given:**

```
Tc2q_max  = 80 ps   (launch FF clock-to-Q, max)
Tc2q_min  = 50 ps   (launch FF clock-to-Q, min)
Tlogic_max = 500 ps  (combinational delay, max path)
Tlogic_min = 100 ps  (combinational delay, min path)
Tsetup    = 50 ps   (capture FF setup requirement)
Thold     = -10 ps  (capture FF hold requirement, negative)
Tskew     = +30 ps  (capture clock arrives 30ps LATER than launch clock)
Tjitter   = 20 ps
```

**Setup analysis (worst case = max delays):**

```
Data arrival time  = T_launch_clk + Tc2q_max + Tlogic_max
                   = 0 + 80 + 500 = 580 ps

Data required time = T_capture_clk + Tperiod - Tsetup - Tjitter
                   = 30 + Tperiod - 50 - 20
                   = Tperiod - 40 ps

For zero slack: 580 = Tperiod - 40
                Tperiod_min = 620 ps
                Fmax = 1/620ps = 1.613 GHz
```

**Hold analysis (best case = min delays):**

```
Data arrival time  = T_launch_clk + Tc2q_min + Tlogic_min
                   = 0 + 50 + 100 = 150 ps

Data required time = T_capture_clk + Thold + Tjitter_hold
                   = 30 + (-10) + 10 = 30 ps

Hold slack = 150 - 30 = 120 ps  (PASS, comfortable margin)
```

**Key observations:**

- Setup depends on clock period. Hold does NOT.
- Positive skew (+30ps) HELPS setup (more time for data) but HURTS hold (less margin).
- Negative hold time (-10ps) is a gift from the cell designer.

### 3.6 Recovery and Removal

These are setup/hold analogs for **asynchronous signals** (reset, set, preset, clear).

**Recovery:** The async signal (e.g., reset deassertion) must be stable at least Trecovery BEFORE the clock edge. If violated, the FF may go metastable.

**Removal:** The async signal must remain stable at least Tremoval AFTER the clock edge.

```
Recovery check (analogous to setup):
  Slack = Tperiod - Tc2q(reset_source) - Tcomb - Trecovery - Tuncertainty

Removal check (analogous to hold):
  Slack = Tc2q_min(reset_source) + Tcomb_min - Tremoval - Tuncertainty_hold
```

Recovery/removal violations on reset are the root cause of many silicon bugs. This is why **asynchronous reset assertion / synchronous reset deassertion** is mandatory.

---

## 4. Reading a PrimeTime Timing Report

### 4.1 Annotated Setup Check Report

```
  Startpoint: u_core/alu_reg[7] (rising edge-triggered flip-flop clocked by sys_clk)
  Endpoint:   u_core/wb_reg[3]  (rising edge-triggered flip-flop clocked by sys_clk)
  Path Group: sys_clk
  Path Type:  max (setup check)

  Point                                    Incr       Path
  -------------------------------------------------------------------
  clock sys_clk (rise edge)                0.000      0.000       <- Launch clock edge at t=0
  clock network delay (propagated)         0.432      0.432       <- Clock tree latency to launch FF
  u_core/alu_reg[7]/CK (DFFRHQX1)         0.000      0.432       <- Clock arrives at launch FF
  u_core/alu_reg[7]/Q (DFFRHQX1)          0.083      0.515       <- Tc2q = 83ps
  u_core/u_alu/g127/Y (NAND2X1)           0.045      0.560       <- First gate in data path
  u_core/u_alu/g128/Y (NOR2X2)            0.038      0.598       <- Second gate
  u_core/u_alu/g129/Y (AOI22X1)           0.052      0.650       <- Third gate
  u_core/u_mux/g42/Y (MUX2X1)             0.041      0.691       <- MUX in data path
  u_core/wb_reg[3]/D (DFFRHQX1)           0.012      0.703       <- Wire delay to capture FF D pin
  data arrival time                                   0.703       <- TOTAL data arrival = 703ps

  clock sys_clk (rise edge)                1.000      1.000       <- Capture clock edge (1 period)
  clock network delay (propagated)         0.448      1.448       <- Clock tree latency to capture FF
  clock reconvergence pessimism            -0.015     1.433       <- CRPR adjustment (see Section 6)
  u_core/wb_reg[3]/CK (DFFRHQX1)          0.000      1.433       <- Clock arrives at capture FF
  library setup time                       -0.048     1.385       <- Setup time = 48ps subtracted
  data required time                                  1.385       <- TOTAL data required = 1385ps

  -------------------------------------------------------------------
  slack (MET)                                         0.682       <- Slack = 1385 - 703 = 682ps
```

**Reading the report step by step:**

1. **Startpoint/Endpoint**: Launch FF and capture FF. Both clocked by sys_clk = same-clock domain analysis.
2. **Clock network delay**: 432ps to launch FF, 448ps to capture FF. Difference = +16ps skew (capture later -> helps setup).
3. **Tc2q**: 83ps from CK to Q of launch FF.
4. **Data path**: Four gates + MUX + wire = 188ps combinational delay.
5. **Data arrival**: 432 + 83 + 188 = 703ps.
6. **CRPR**: -15ps removed from capture clock (see Section 6). This tightens the capture time.
7. **Setup requirement**: 48ps subtracted from capture clock arrival.
8. **Data required**: 1433 - 48 = 1385ps.
9. **Slack**: 1385 - 703 = 682ps. Positive = MET.

### 4.2 Hold Check Report

```
  Path Type: min (hold check)

  Point                                    Incr       Path
  -------------------------------------------------------------------
  clock sys_clk (rise edge)                0.000      0.000
  clock network delay (propagated)         0.401      0.401       <- Min clock delay to launch FF
  u_core/alu_reg[7]/CK (DFFRHQX1)         0.000      0.401
  u_core/alu_reg[7]/Q (DFFRHQX1)          0.058      0.459       <- Tc2q_MIN = 58ps
  u_core/u_alu/g127/Y (NAND2X1)           0.031      0.490       <- Min delays through logic
  u_core/wb_reg[3]/D (DFFRHQX1)           0.008      0.498
  data arrival time                                   0.498

  clock sys_clk (rise edge)                0.000      0.000       <- SAME edge (hold = same-edge check)
  clock network delay (propagated)         0.465      0.465       <- Max clock delay to capture FF
  clock reconvergence pessimism            0.012      0.477       <- CRPR for hold
  u_core/wb_reg[3]/CK (DFFRHQX1)          0.000      0.477
  library hold time                        -0.012     0.465       <- Hold = -12ps (negative!)
  data required time                                  0.465

  -------------------------------------------------------------------
  slack (MET)                                         0.033       <- 498 - 465 = 33ps margin
```

**Key differences from setup report:**

- Uses **min** delays for data path (fastest data can arrive).
- Uses **max** clock delay for capture path (latest the capture clock can arrive -- worst case for hold).
- Hold check is against the **same** clock edge (time = 0, not Tperiod).
- Negative hold time (-12ps) effectively relaxes the requirement.

---

## 5. Multi-Cycle Paths (MCP)

### 5.1 The Concept

A multi-cycle path is a path where the designer knows data takes N clock cycles to propagate, and the capture FF only samples every N cycles (controlled by an enable signal).

### 5.2 Detailed Example: 2-Cycle MCP at 500 MHz (Tperiod = 2ns)

```
Waveform:
  CLK:    |‾|_|‾|_|‾|_|‾|_|‾|_|
  Time:   0  1  2  3  4  5  6  7  8  9  10 (ns)
  Edges:  E0    E1    E2    E3    E4

Without MCP:
  Launch: E0 (t=0)
  Setup check: E1 (t=2ns)   <- Only 1 period budget
  Hold check:  E0 (t=0)     <- Same edge as launch

With set_multicycle_path 2 -setup:
  Launch: E0 (t=0)
  Setup check: E2 (t=4ns)   <- 2 periods budget (relaxed)
  Hold check:  E0 (t=0)     <- DEFAULT: still at E0 (WRONG for most cases!)

With set_multicycle_path 2 -setup AND set_multicycle_path 1 -hold:
  Launch: E0 (t=0)
  Setup check: E2 (t=4ns)   <- 2 periods budget
  Hold check:  E1 (t=2ns)   <- Moved forward by 1 period (CORRECT!)
```

### 5.3 Why Hold MCP = Setup MCP - 1 (Proof by Timing Diagram)

```
  CLK:    _|‾‾|__|‾‾|__|‾‾|__|‾‾|__
  Edge:    E0      E1      E2      E3

  Setup MCP = 2:
    Data launched at E0, must arrive by E2 - Tsu
    The NEXT launch is at E1 (when enable fires again? NO!)

    Actually, the physical constraint is:
    Data launched at E0 must NOT corrupt the capture at E1
    (which would be capturing the PREVIOUS cycle's data)

  If we leave hold at default (check at E0):
    Hold check: data from E0 must arrive AFTER E0 + Th
    This is trivially met (Tc2q > Th almost always)
    But it forces the tool to hold-fix against E0, potentially
    inserting HUGE buffers that break setup.

  If we set hold MCP = 1 (check at E1):
    Hold check: data from E0 must arrive AFTER E1 + Th = 2ns + Th
    This is the REAL constraint -- data from E0's launch must not
    arrive so late that it violates hold at E1's capture.

    Since we gave 4ns for setup and data arrives around 3-3.5ns,
    the hold at E1 (2ns) is naturally met.
```

**The rule:** For setup MCP = N, set hold MCP = N-1. This moves the hold check edge to (N-1) periods after the launch edge.

### 5.4 SDC for Common MCP Scenarios

```tcl
# 2-cycle multiply result (enable pulses every 2 clocks)
set_multicycle_path 2 -setup -from [get_pins mult_reg/Q] -to [get_pins result_reg/D]
set_multicycle_path 1 -hold  -from [get_pins mult_reg/Q] -to [get_pins result_reg/D]

# Slow-to-fast domain (clk_slow = 100MHz, clk_fast = 200MHz, ratio 1:2)
# Data launched by slow clock is sampled 2 fast edges later
set_multicycle_path 2 -setup -from [get_clocks clk_slow] -to [get_clocks clk_fast]
set_multicycle_path 1 -hold  -from [get_clocks clk_slow] -to [get_clocks clk_fast]

# 4-stage pipeline accumulator (enable every 4 cycles)
set_multicycle_path 4 -setup -from [get_cells accum_stage1*] -to [get_cells accum_stage4*]
set_multicycle_path 3 -hold  -from [get_cells accum_stage1*] -to [get_cells accum_stage4*]
```

### 5.5 MCP Pitfalls

1. **Forgetting the hold MCP**: Leaves hold check at the launch edge, causing massive hold buffer insertion that degrades setup.
2. **Setting MCP on CDC paths**: Use `set_false_path` instead. MCP implies a known phase relationship.
3. **MCP on paths with multiple endpoints**: Verify each endpoint actually requires the MCP.
4. **MCP with negative skew**: The hold constraint becomes harder; verify with your specific skew values.

---

## 6. Clock Reconvergence Pessimism Removal (CRPR/CPPR)

### 6.1 The Problem

In OCV analysis, the launch and capture clock paths are derated in opposite directions:

- **Setup**: Launch path uses max delay, capture path uses min delay (pessimistic worst case).
- **Hold**: Launch path uses min delay, capture path uses max delay.

But if the launch and capture clock paths share common segments (reconverge from the same clock root), the shared portion CANNOT simultaneously be fast AND slow. The tool is being overly pessimistic by applying opposite derates to the same physical buffer.

### 6.2 Example

```
                     CLK_ROOT
                        |
                    [BUF_common]  <- This buffer is shared
                     /        \
                [BUF_L]      [BUF_C]
                   |            |
               Launch_FF    Capture_FF
```

**Without CRPR (setup analysis):**

```
Launch clock path (max derate = 1.05):
  BUF_common delay: 100ps * 1.05 = 105ps
  BUF_L delay:       80ps * 1.05 =  84ps
  Total launch:                    = 189ps

Capture clock path (min derate = 0.95):
  BUF_common delay: 100ps * 0.95 =  95ps
  BUF_C delay:       90ps * 0.95 =  85.5ps
  Total capture:                   = 180.5ps

Skew (capture - launch) = 180.5 - 189 = -8.5ps  (hurts setup)
```

**With CRPR:**

BUF_common cannot be both 105ps (launch) and 95ps (capture). The CRPR credit equals the OCV difference on the common segment:

```
CRPR = BUF_common_max - BUF_common_min = 105 - 95 = 10ps

Adjusted capture clock = 180.5 + 10 = 190.5ps
Adjusted skew = 190.5 - 189 = +1.5ps  (now slightly helps setup)

CRPR recovered 10ps of pessimism!
```

### 6.3 CRPR in PrimeTime

```
  clock reconvergence pessimism    -0.015    1.433
```

This line appears in every timing report. The tool automatically identifies the common clock path portion and removes the OCV pessimism from it.

**CRPR is larger when:**
- Clock paths share more common buffers (long common path, short divergence).
- OCV derate is large.
- Typical values: 5-50ps depending on design.

### 6.4 Enabling CRPR

```tcl
# In PrimeTime
set timing_remove_clock_reconvergence_pessimism true
# or
set si_enable_analysis true  ; # SI mode automatically enables CRPR
```

---

## 7. OCV / AOCV / POCV Deep Dive

### 7.1 Flat OCV (On-Chip Variation)

OCV models within-die process variation by applying a flat derate to all cell delays.

```
For setup analysis:
  Launch path (early): delay * (1 - derate)    e.g., * 0.95
  Capture path (late):  delay * (1 + derate)    e.g., * 1.05

For hold analysis (reversed):
  Launch path (late):  delay * (1 + derate)
  Capture path (early): delay * (1 - derate)
```

Typical OCV derate values:

| Node  | OCV Derate |
|-------|-----------|
| 65nm  | 3-5%      |
| 28nm  | 5-8%      |
| 16nm  | 8-12%     |
| 7nm   | 10-15%    |

**Problem:** Flat OCV is overly pessimistic for deep paths. A path through 50 gates is very unlikely to have ALL gates be 5% slow simultaneously. Statistical averaging means the variation on a deep path is much less than the worst-case derate.

### 7.2 AOCV (Advanced OCV)

AOCV uses a derate table indexed by **path depth** (number of cells in the path) and optionally **physical distance**.

```
/* AOCV derate table example */
aocv_table (early_derate) {
  /* depth:    1      2      3      5      10     20     50 */
  values ("0.920, 0.935, 0.945, 0.955, 0.965, 0.975, 0.985");
}

aocv_table (late_derate) {
  /* depth:    1      2      3      5      10     20     50 */
  values ("1.080, 1.065, 1.055, 1.045, 1.035, 1.025, 1.015");
}
```

**Depth = 1 (single cell):** Large derate (8% early, 8% late) -- full variation possible.
**Depth = 50 (deep path):** Small derate (1.5% early, 1.5% late) -- variation averages out.

**Numerical example:**

```
Path with 20 cells, each with nominal delay 50ps:
  Nominal path delay = 20 * 50ps = 1000ps

  With flat OCV (5%):
    Late path delay = 1000 * 1.05 = 1050ps  (50ps pessimism)

  With AOCV (depth=20, late derate=1.025):
    Late path delay = 1000 * 1.025 = 1025ps  (25ps pessimism)

  AOCV saved 25ps of pessimism!
```

### 7.3 POCV (Parametric OCV) / SOCV (Statistical OCV)

POCV models each cell's delay as a Gaussian random variable:

```
Cell delay = mean + k * sigma

Where:
  mean  = nominal delay from .lib
  sigma = standard deviation (from POCV .lib extension)
  k     = number of sigma for coverage target
```

**Path delay statistics:**

```
Path mean  = sum(mean_i)    for all cells i in the path
Path sigma = sqrt(sum(sigma_i^2))    (independent random variables)
```

Note: sigma adds in **quadrature** (root-sum-of-squares), NOT linearly. This is the key insight -- for a path with N identical cells:

```
Cell sigma = s
Path sigma = s * sqrt(N)    (NOT s * N)

Path mean  = N * mean
Path 3-sigma delay = N * mean + 3 * s * sqrt(N)

Flat OCV equivalent = N * mean * (1 + derate) = N * mean + N * mean * derate
```

For large N, `3*s*sqrt(N) << N*mean*derate`, proving POCV is much less pessimistic.

**POCV in PrimeTime:**

```tcl
set timing_pocvm_enable_analysis true
read_parasitics -format spef design.spef
# POCV data in .lib:
# cell (INVX1) {
#   pocv_sigma_cell_rise (...) { ... }
#   pocv_sigma_cell_fall (...) { ... }
# }
```

**Practical impact:**

| Method | Slack on a 50-cell path (example) |
|--------|-----------------------------------|
| Flat OCV (5%) | -35ps (failing) |
| AOCV | +12ps (passing) |
| POCV (3-sigma) | +28ps (comfortable) |

Moving from flat OCV to POCV can recover 50-80ps of slack on deep paths, which translates to either higher Fmax or smaller area (fewer upsized cells).

---

## 8. Crosstalk and Signal Integrity (SI) in STA

### 8.1 Capacitive Coupling Model

Adjacent wires in the same metal layer (or adjacent layers) create coupling capacitance.

```
          Wire A (aggressor)
  ═══════════════════════════
          Cc (coupling cap)
  ───────────────────────────
          Wire B (victim)
```

The victim wire sees an effective capacitance that depends on the aggressor's switching:

```
When aggressor switches in the SAME direction as victim:
  C_eff = C_ground + Cc * (1 - Kswitch)    <- Cc partially "disappears"
  Delay DECREASES (speeds up)  -- "helpful crosstalk"

When aggressor switches in the OPPOSITE direction:
  C_eff = C_ground + Cc * (1 + Kswitch)    <- Cc effectively doubles
  Delay INCREASES (slows down)  -- "harmful crosstalk"

When aggressor is quiet:
  C_eff = C_ground + Cc               <- Normal capacitance

Kswitch: switching factor, typically 0.5-2.0 depending on relative slew
```

### 8.2 Delta Delay Calculation

```
Delta delay = delay_with_crosstalk - delay_without_crosstalk

For a victim net with coupling cap Cc to an aggressor:
  Worst case delta = Cc * (V_aggressor / V_victim) * R_victim

Simplified Miller factor approach:
  Same-direction switching:  effective Cc = Cc * (1 - slew_ratio)
  Opposite-direction switching: effective Cc = Cc * (1 + slew_ratio)

  where slew_ratio approximates the temporal overlap
```

**Numerical example:**

```
Victim wire: R = 50 ohm, C_ground = 20 fF
Coupling cap to aggressor: Cc = 15 fF
Aggressor switches in opposite direction (harmful)

Without crosstalk:
  delay ~ 0.69 * R * C_total = 0.69 * 50 * 35fF = 1.21 ps

With crosstalk (Miller factor = 2x on Cc):
  C_effective = 20 + 2*15 = 50 fF
  delay ~ 0.69 * 50 * 50fF = 1.73 ps

Delta delay = 0.52 ps  (43% increase!)
```

At advanced nodes with tight pitch (7nm: ~28nm pitch), coupling capacitance can be 50-70% of total wire capacitance, making SI-aware STA critical.

### 8.3 Glitch Analysis

A glitch occurs when the aggressor switches but the victim is quiet. The coupling injects a voltage bump on the victim.

```
Glitch height = Cc / (Cc + C_ground + C_gate) * V_DD * f(slew_ratio)
```

**Glitch classification:**

| Glitch Height  | Risk Level | Action Required |
|----------------|-----------|-----------------|
| < 0.3 * VDD   | Safe      | None            |
| 0.3-0.5 * VDD | Warning   | May propagate   |
| > 0.5 * VDD   | Fail      | Will be captured by downstream gate as logic transition |

Glitches on clock nets are especially dangerous -- they can cause double-clocking.

### 8.4 SI-Aware STA in PrimeTime SI

```tcl
# Enable SI analysis
set si_enable_analysis true
set si_xtalk_reselect_delta_delay 0.005  ; # reselect threshold

# Crosstalk delay report
report_timing -crosstalk_delta

# Report shows delta delay per net:
#   u_core/net123 (delta = +0.015ns)  <- crosstalk added 15ps
```

PrimeTime SI considers:

1. **Timing windows** of aggressors -- only aggressors that switch in a temporal window near the victim's transition are considered (reduces pessimism).
2. **Multiple aggressors** -- can compound.
3. **Functional correlation** -- if aggressor and victim always switch in the same direction (functionally correlated), only helpful crosstalk applies. Requires CPPR-like analysis.

### 8.5 Fixing Crosstalk Violations

| Fix | Mechanism | Trade-off |
|-----|-----------|-----------|
| Increase wire spacing | Reduces Cc quadratically | Routing congestion |
| Insert shield wires | Grounded wire between aggressor/victim | Area, routing resources |
| NDR (Non-Default Rules) | Double-space for critical nets | Area |
| Upsize victim driver | Stronger drive reduces sensitivity | Power, area |
| Reroute on different layer | Different layer has different neighbors | May increase wirelength |
| Buffer insertion | Breaks long coupling segment | Area, power |

---

## 9. Timing ECO Flow

### 9.1 What is a Timing ECO?

After signoff STA reveals remaining violations, an **Engineering Change Order** (ECO) makes targeted fixes without re-running the full PnR flow.

### 9.2 Types of ECOs

**Pre-mask ECO (metal-only ECO):**
- Changes only metal layers (no base layer changes).
- Can swap cells of the same height from the spare cell pool.
- Very fast turnaround (days vs. weeks for full respin).

**Functional ECO:**
- Changes logic (new cells, new connections).
- May require spare cells or filler cell replacement.
- More disruptive but still cheaper than respin.

### 9.3 Common Timing ECO Operations

```tcl
# In PrimeTime / ICC2 ECO flow:

# 1. Cell upsizing on critical path
size_cell u_core/g127 NAND2X2    ; # Upsize from X1 to X2
# Effect: Reduces gate delay by ~30-40%, increases area/power slightly

# 2. Vt swapping on critical path
size_cell u_core/g128 NOR2X2_LVT ; # Swap from SVT to LVT
# Effect: Reduces delay by ~15-20%, increases leakage by ~5-10x

# 3. Buffer insertion for hold fixing
insert_buffer u_core/net45 BUFX2 ; # Add buffer on short path
# Effect: Adds ~40ps delay to fix hold violation

# 4. Buffer insertion to reduce fanout
insert_buffer u_core/net78 BUFX4 ; # Drive high-fanout net
# Effect: Reduces load on critical gate, improving delay and slew

# 5. Useful skew insertion
insert_buffer u_core/clk_net CLKBUFX4 ; # Add buffer on clock path
# Effect: Delays clock to capture FF, borrowing time for setup
# WARNING: This is dangerous post-signoff; can break hold on other paths
```

### 9.4 ECO Verification

After ECO, must re-verify:
1. Timing (incremental STA on affected paths)
2. DRC (max transition, max capacitance, min pulse width)
3. LVS (layout vs. schematic)
4. DRC (physical design rules)

---

## 10. Full SDC Walkthrough for a Realistic SoC Block

### 10.1 Scenario: Video Processing SoC Block

```
Block: video_proc
Clocks:
  - sys_clk:    500 MHz (from PLL, jitter 30ps)
  - pixel_clk:  148.5 MHz (from external, jitter 50ps)
  - scan_clk:   50 MHz (DFT scan clock)
  - jtag_tck:   25 MHz (JTAG debug)
  
Interfaces:
  - AXI4 master to DDR controller (sys_clk domain)
  - Pixel input from camera (pixel_clk domain)
  - APB slave for configuration (sys_clk domain)
  - JTAG port
  - Scan chains
```

### 10.2 Complete SDC File

```tcl
###############################################
# Clock Definitions
###############################################

# Primary system clock (500 MHz)
create_clock -name sys_clk -period 2.0 -waveform {0 1.0} [get_ports HCLK]

# Pixel clock from external source (148.5 MHz)
create_clock -name pixel_clk -period 6.734 -waveform {0 3.367} [get_ports PCLK]

# Scan clock (50 MHz)
create_clock -name scan_clk -period 20.0 [get_ports SCAN_CLK]

# JTAG clock (25 MHz)
create_clock -name jtag_tck -period 40.0 [get_ports TCK]

# Generated clock: divide-by-2 from sys_clk (250 MHz internal processing)
create_generated_clock -name proc_clk \
    -source [get_ports HCLK] \
    -divide_by 2 \
    [get_pins u_clkdiv/Q]

# Generated clock: PLL output (locked to sys_clk)
create_generated_clock -name pll_clk \
    -source [get_ports HCLK] \
    -multiply_by 2 \
    -duty_cycle 50 \
    [get_pins u_pll/CLK_OUT]

###############################################
# Clock Uncertainty
###############################################

# Pre-CTS uncertainties (jitter + estimated skew)
set_clock_uncertainty 0.200 -setup [get_clocks sys_clk]
set_clock_uncertainty 0.050 -hold  [get_clocks sys_clk]

set_clock_uncertainty 0.250 -setup [get_clocks pixel_clk]
set_clock_uncertainty 0.060 -hold  [get_clocks pixel_clk]

# Inter-clock uncertainty (sys_clk <-> pixel_clk domain crossing)
set_clock_uncertainty 0.300 -setup \
    -from [get_clocks sys_clk] -to [get_clocks pixel_clk]
set_clock_uncertainty 0.300 -setup \
    -from [get_clocks pixel_clk] -to [get_clocks sys_clk]

# Scan clock (relaxed)
set_clock_uncertainty 0.100 -setup [get_clocks scan_clk]
set_clock_uncertainty 0.050 -hold  [get_clocks scan_clk]

###############################################
# Clock Latency (pre-CTS only; removed post-CTS)
###############################################

set_clock_latency 0.800 -source [get_clocks sys_clk]      ; # PLL source latency
set_clock_latency 1.200 [get_clocks sys_clk]               ; # Estimated network latency

###############################################
# Clock Groups (async domain declaration)
###############################################

set_clock_groups -asynchronous \
    -group [get_clocks {sys_clk proc_clk pll_clk}] \
    -group [get_clocks pixel_clk] \
    -group [get_clocks scan_clk] \
    -group [get_clocks jtag_tck]

###############################################
# I/O Constraints
###############################################

# AXI master output (to DDR controller)
set_output_delay -clock sys_clk -max 0.800 [get_ports {AXI_AW* AXI_W* AXI_AR*}]
set_output_delay -clock sys_clk -min 0.100 [get_ports {AXI_AW* AXI_W* AXI_AR*}]

# AXI master input (from DDR controller)
set_input_delay -clock sys_clk -max 1.200 [get_ports {AXI_R* AXI_B*}]
set_input_delay -clock sys_clk -min 0.200 [get_ports {AXI_R* AXI_B*}]

# Pixel input (camera interface, DDR capture on pixel_clk)
set_input_delay -clock pixel_clk -max 2.500 [get_ports {PIX_DATA* PIX_VSYNC PIX_HSYNC}]
set_input_delay -clock pixel_clk -min 0.500 [get_ports {PIX_DATA* PIX_VSYNC PIX_HSYNC}]

# APB slave interface (from upstream AHB-APB bridge)
set_input_delay -clock sys_clk -max 1.000 [get_ports {PADDR* PWDATA* PSEL PENABLE PWRITE}]
set_input_delay -clock sys_clk -min 0.150 [get_ports {PADDR* PWDATA* PSEL PENABLE PWRITE}]
set_output_delay -clock sys_clk -max 0.600 [get_ports {PRDATA* PREADY PSLVERR}]
set_output_delay -clock sys_clk -min 0.100 [get_ports {PRDATA* PREADY PSLVERR}]

# JTAG (relaxed timing -- 25 MHz)
set_input_delay -clock jtag_tck -max 15.0 [get_ports {TDI TMS}]
set_output_delay -clock jtag_tck -max 15.0 [get_ports TDO]

###############################################
# Timing Exceptions
###############################################

# Static configuration registers (written once at boot, read continuously)
set_false_path -from [get_cells u_apb_regs/cfg_reg*]

# Asynchronous reset (handled by reset synchronizers)
set_false_path -from [get_ports RST_N]

# Multi-cycle path: 4-tap FIR filter accumulation (result valid every 4 cycles)
set_multicycle_path 4 -setup \
    -from [get_cells u_fir/tap_reg*] -to [get_cells u_fir/accum_reg*]
set_multicycle_path 3 -hold \
    -from [get_cells u_fir/tap_reg*] -to [get_cells u_fir/accum_reg*]

# Multi-cycle path: multiplier output (2-cycle)
set_multicycle_path 2 -setup \
    -from [get_cells u_mul/a_reg*] -to [get_cells u_mul/p_reg*]
set_multicycle_path 1 -hold \
    -from [get_cells u_mul/a_reg*] -to [get_cells u_mul/p_reg*]

# False path through test MUX in functional mode
set_false_path -through [get_pins u_dft_mux/TEST_SEL]

###############################################
# Design Rule Constraints
###############################################

set_max_transition 0.250 [current_design]        ; # 250ps max slew
set_max_capacitance 0.100 [current_design]        ; # 100fF max load
set_max_fanout 32 [current_design]

# Tighter for clock nets
set_max_transition 0.150 [get_clocks sys_clk]
set_max_transition 0.200 [get_clocks pixel_clk]

###############################################
# Operating Conditions
###############################################

set_operating_conditions -max ss_0p72v_125c -min ff_0p88v_m40c
# For MCMM: operating conditions set per scenario in the analysis script
```

---

## 11. Clock Tree Synthesis (CTS)

### 11.1 CTS Goals and Metrics

| Metric | Target (typical SoC) | Why |
|--------|---------------------|-----|
| Skew | < 50-100ps | Directly affects timing margin |
| Latency | 500-1500ps | Affects cycle time through launch/capture |
| Max transition | < 150-250ps | Signal integrity, downstream gate delay |
| Power | Minimize | Clock tree = 30-50% of total dynamic power |
| Insertion delay variation | < 20ps (3-sigma) | OCV/POCV margin |

### 11.2 CTS Topologies

**H-tree:** Perfectly symmetric binary tree. Wire lengths balanced by construction. Used for clock meshes in memory arrays and FPGAs. Impractical for irregular ASIC layouts.

**Fishbone/Spine:** Central spine with branches. Good for row-based standard cell designs. Less balanced than H-tree but adapts to irregular placement.

**Balanced buffer tree:** What most CTS tools build. Algorithm: cluster sinks by proximity, insert buffers, balance delay through buffer sizing and wire snaking.

**Clock mesh/grid:** A grid of wires shorts the clock signal at regular intervals. Lowest skew possible (< 10ps) but highest power. Used for very high-frequency designs (server CPUs at 5+ GHz).

### 11.3 CTS Design Rules

```tcl
# In ICC2 / Innovus CTS constraints:

# Use only dedicated clock buffers/inverters
set_clock_tree_references -references {CLKBUFX4 CLKBUFX8 CLKBUFX16 CLKINVX4 CLKINVX8}

# Route clocks on upper metal layers (low resistance)
set_clock_routing_rules -rule NDR_2W2S -layer {M4 M5 M6}

# Non-Default Routing Rules for clock nets
create_routing_rule NDR_2W2S \
    -widths {M4 0.08 M5 0.08 M6 0.08} \
    -spacings {M4 0.08 M5 0.08 M6 0.08}
# Double-width, double-space reduces resistance and crosstalk

# Shield clock nets
set_clock_tree_options -use_shield_net VSS
```

### 11.4 Useful Skew in CTS

```
Before useful skew:
  Path A->B: slack = -50ps (FAIL)
  Path C->B: slack = +200ps (comfortable)

CTS intentionally delays B's clock by 50ps:
  Path A->B: slack = -50 + 50 = 0ps (MET)
  Path C->B: slack = +200 - 50 = +150ps (still fine)
```

The CTS tool solves a global optimization problem: minimize total negative slack across all paths by redistributing skew within the clock tree. Constraint: no path may go negative after skew redistribution.

### 11.5 Post-CTS Timing Closure

```
Post-CTS signoff checklist:
1. set_propagated_clock [all_clocks]     ; # Use real clock tree
2. Reduce set_clock_uncertainty (remove skew component, keep jitter)
3. Run STA -> identify setup violations
4. Fix setup: cell sizing, Vt swap on data paths
5. Run STA -> identify hold violations
6. Fix hold: buffer insertion on short data paths
7. Iterate until clean at all corners
8. Verify CTS DRC: max_transition, max_capacitance on clock nets
```

---

## 12. MCMM and Signoff Methodology

### 12.1 Multi-Corner Multi-Mode

**Corners** = PVT (Process, Voltage, Temperature) operating conditions.
**Modes** = Functional states (functional, scan, sleep, etc.)

```
Typical signoff scenario matrix:

| Corner | Process | Voltage | Temp  | Check       |
|--------|---------|---------|-------|-------------|
| SS_LV  | Slow    | 0.72V   | 125C  | Setup (max) |
| SS_LV_LT| Slow   | 0.72V   | -40C  | Setup (inv) |
| FF_HV  | Fast    | 0.88V   | -40C  | Hold (min)  |
| FF_HV_HT| Fast   | 0.88V   | 125C  | Hold (inv)  |
| TT_NV  | Typical | 0.80V   | 25C   | Nominal     |

Modes: functional, scan_shift, scan_capture, sleep, MBIST

Total scenarios = 5 corners x 4 modes = 20 scenarios
Each scenario has its own .lib, SPEF, and SDC
```

**Temperature inversion** (at advanced nodes < 28nm): At low temperature, mobility increases (cells should be faster), but threshold voltage also increases due to reduced thermal voltage. At low Vdd (near Vth), the Vth increase dominates, making cells SLOWER at low temperature. This is why we check setup at BOTH high and low temperature.

### 12.2 Signoff Criteria

```
All of the following must be clean at ALL signoff corners/modes:

1. Setup slack >= 0        (all paths)
2. Hold slack >= 0         (all paths)
3. Recovery slack >= 0     (all async paths)
4. Removal slack >= 0      (all async paths)
5. Max transition clean    (no slew violations)
6. Max capacitance clean   (no overloaded nets)
7. Min pulse width clean   (clock pulse wide enough for FFs)
8. Clock gating checks     (setup/hold on ICG enable)
9. No noise (SI) failures  (glitch height within limits)
```

---

## 13. Interview Questions & Answers

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
