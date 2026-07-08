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
13. .lib Format Deep-Dive
14. SPEF Walkthrough
15. Clock Gating Checks

---

## 1. Cell Delay Modeling

### 1.1 NLDM (Non-Linear Delay Model)

NLDM is the traditional .lib delay model used from 180nm through 28nm. Each timing arc has a 2D lookup table indexed by:

- **Input transition time** (input slew, rise or fall)
- **Output load capacitance** (total capacitance seen at output pin)

The table stores **cell delay** and **output slew** values.

```text
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

```text
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

```text
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

```text
For a single RC segment: T_Elmore = R * C
For distributed RC:      T_Elmore = sum(R_i * C_downstream_i)
```

**Arnoldi/AWE (Asymptotic Waveform Evaluation):**

More accurate than Elmore for multi-segment RC networks. Used in signoff tools with SPEF extraction.

**SPEF (Standard Parasitic Exchange Format):**

```text
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

```text
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

```text
Buffer:  A rises -> Y rises    (positive unate)
         A falls -> Y falls

AND gate (from A, with B=1):
         A rises -> Y rises    (positive unate on A)
         A falls -> Y falls
```

**Negative unate:** Output transitions in the OPPOSITE direction.

```text
Inverter:  A rises -> Y falls  (negative unate)
           A falls -> Y rises

NAND gate (from A, with B=1):
           A rises -> Y falls  (negative unate on A)
           A falls -> Y rises
```

**Non-unate:** Output direction depends on OTHER input values.

```text
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

```text
timing () {
  related_pin: "A";
  timing_sense: positive_unate;  /* or negative_unate, non_unate */
}
```

### 2.3 Timing Arc Conditions

Conditional arcs model data-dependent delays (e.g., different delays through a MUX depending on select):

```text
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

## 3. Setup and Hold -- Complete Derivation from Scratch

### 3.1 Deriving Setup and Hold Slack from Path Delays

```ascii-graph
  Consider a timing path from Launch FF to Capture FF:
  
  CLK_SRC ──[clk_path_launch]──> Launch_FF ──[data_path]──> Capture_FF <──[clk_path_capture]── CLK_SRC
                                       |                          |
                                    CK -> Q                    D pin
                                    delay                       input

  Launch path (data arrival):
  ═════════════════════════════
  Data arrival time = T_clk_launch + T_c2q + T_combo
  where:
    T_clk_launch = delay from clock source to Launch_FF clock pin
    T_c2q        = clock-to-Q delay of Launch FF
    T_combo      = combinational logic delay from Q to D

  Capture path (data required for setup):
  ═══════════════════════════════════════════
  Data required time = T_clk_capture + T_period - T_setup - T_uncertainty
  where:
    T_clk_capture = delay from clock source to Capture_FF clock pin
    T_period      = clock period
    T_setup       = setup time requirement of Capture FF
    T_uncertainty = clock uncertainty (jitter + margin)

  Setup Slack:
  ══════════
  Setup_slack = Data_required - Data_arrival
              = (T_clk_capture + T_period - T_setup - T_uncertainty)
                - (T_clk_launch + T_c2q + T_combo)
              = T_period + (T_clk_capture - T_clk_launch) - T_c2q - T_combo - T_setup - T_uncertainty
              = T_period + T_skew - T_c2q - T_combo - T_setup - T_uncertainty

  where T_skew = T_clk_capture - T_clk_capture (capture clock delay minus launch clock delay)

  Setup slack must be >= 0. If negative, the path violates setup timing.

  Capture path (data required for hold):
  ══════════════════════════════════════
  Hold check is against the SAME clock edge (not next edge):
  Hold data required = T_clk_capture + T_hold
  Hold data arrival  = T_clk_launch + T_c2q_min + T_combo_min

  Hold Slack:
  ══════════
  Hold_slack = Data_arrival_min - Data_required_hold
             = (T_clk_launch_min + T_c2q_min + T_combo_min)
               - (T_clk_capture_max + T_hold)
             = (T_clk_launch_min - T_clk_capture_max) + T_c2q_min + T_combo_min - T_hold
             = -T_skew_hold + T_c2q_min + T_combo_min - T_hold

  Note: for hold, clock skew works AGAINST you (positive skew helps setup,
  hurts hold; negative skew helps hold, hurts setup).
```

### 3.2 Worked Timing Diagram with All Components

```ascii-graph
  Given:
    T_period = 1.0 ns (1 GHz)
    T_clk_launch  = 0.40 ns (max), 0.35 ns (min)
    T_clk_capture = 0.45 ns (max), 0.38 ns (min)
    T_c2q         = 0.08 ns (max), 0.05 ns (min)
    T_combo       = 0.35 ns (max), 0.15 ns (min)
    T_setup       = 0.05 ns
    T_hold        = 0.02 ns (positive hold time)
    T_uncertainty = 0.10 ns (setup), 0.03 ns (hold)

  ─── SETUP ANALYSIS (use MAX delays for data path, skew helps/hurts) ───

  Complete timing diagram (all times in ns):

  CLK_SRC:  0.00                                          1.00
            ┌──┐                                          ┌──┐
            │  │                                          │  │
            └──┘                                          └──┘
            t=0                                         t=1.0

  Launch clock arrives at Launch_FF (CLK + T_clk_launch_max):
            ────── 0.40 ──────┐
                               └──> Launch_FF CK pin

  Launch_FF Q output valid (CLK + T_clk_launch + T_c2q_max):
            ────── 0.48 ──────┐
                               └──> data starts propagating

  Data propagates through combo logic (T_combo_max):
            ────── 0.83 ──────┐
                               └──> data arrives at Capture_FF D pin

                                    DATA ARRIVAL = 0.83 ns

  Capture clock arrives at Capture_FF (CLK + T_period + T_clk_capture_max):
            ────── 1.45 ──────┐
                               └──> Capture_FF CK pin
                               |<T_setup>|
                               1.40       1.45
                               |<T_unc>|
                               1.35      1.45

  Data required time calculation:
    Capture clock edge at source = 1.00 ns (next edge)
    + T_clk_capture_max          = 0.45 ns
    - T_setup                    = 0.05 ns
    - T_uncertainty_setup        = 0.10 ns
    ─────────────────────────────────────
    Data required                 = 1.30 ns

  Data arrival                   = 0.83 ns
  Setup slack = 1.30 - 0.83     = 0.47 ns  PASS

  ─── HOLD ANALYSIS (use MIN delays for data, MAX for capture clock) ───

  Hold checks the SAME clock edge (t=0), not the next edge:

  CLK_SRC:  0.00
            ┌──┐
            │  │
            └──┘
            t=0

  Launch clock arrives (CLK + T_clk_launch_min):
            ────── 0.35 ──────┐
                               └──> Launch_FF CK pin

  Launch_FF Q output valid (min delays):
            ────── 0.40 ──────┐
                               └──> data starts propagating (0.35+0.05)

  Data through combo logic (min):
            ────── 0.55 ──────┐
                               └──> data arrives at Capture_FF D pin

                                    DATA ARRIVAL (min) = 0.55 ns

  Capture clock arrives (CLK + T_clk_capture_max, same edge):
            ────── 0.45 ──────┐
                               └──> Capture_FF CK pin
                               |<-T_hold->|
                               0.45       0.47

  Data required for hold:
    Capture clock arrival = T_clk_capture_max = 0.45 ns
    + T_hold              = 0.02 ns
    + T_uncertainty_hold  = 0.03 ns
    ─────────────────────────────────
    Data required          = 0.50 ns

  Data arrival (min)      = 0.55 ns
  Hold slack = 0.55 - 0.50 = 0.05 ns  PASS

  Summary:
    Setup slack = +0.47 ns (comfortable margin)
    Hold slack  = +0.05 ns (tight but passing)

  Observation: positive skew (capture clock later than launch) HELPS setup
  (data has more time to arrive) but HURTS hold (data must be stable longer
  past the later capture edge).

  Common path: the portion of the clock path from source to the divergence
  point where launch and capture paths split. This shared segment has the
  SAME delay in reality, but OCV analysis derates it differently for launch
  and capture -- this is the source of CRPR pessimism (see Section 6).
```

### 3.3 Inside a Master-Slave Edge-Triggered Flip-Flop

A standard positive-edge-triggered D-FF consists of two transmission-gate latches:

```ascii-graph
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

### 3.4 Setup Time -- Physical Meaning

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

```ascii-graph
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

### 3.5 Hold Time -- Physical Meaning

**Hold time** is the minimum time D must remain stable AFTER the clock edge so that TG1 fully disconnects the input from M_node before D changes.

Physically, TG1 does not close instantaneously -- it takes a finite time for the clock to propagate through the local clock inverter and fully turn off the transmission gate. If D changes during this window, the new D value "leaks" into M_node and corrupts the latched data.

### 3.6 Negative Hold Time

Modern flip-flops in standard cell libraries often have negative hold time (e.g., -20ps). This means D can change BEFORE the clock edge and still be correctly captured. This occurs when:

1. The internal clock path to TG1 is slower than the external data path to the TG1 input.
2. The clock must propagate through internal inverters before it reaches TG1's control.
3. D can change up to |Th| before the external clock edge because TG1 hasn't received the clock transition yet.

Negative hold time is extremely desirable -- it makes hold timing easier to close.

### 3.5 Setup/Hold Analysis -- Full Numerical Walkthrough

**Given:**

```text
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

- **Data arrival time** = `T_launch_clk + Tc2q_max + Tlogic_max`
   - = 0 + 80 + 500 = 580 ps

- **Data required time** = `T_capture_clk + Tperiod - Tsetup - Tjitter`
   - = 30 + Tperiod - 50 - 20
   - = Tperiod - 40 ps

For zero slack: 580 = Tperiod - 40
Tperiod_min = 620 ps
Fmax = 1/620ps = 1.613 GHz

**Hold analysis (best case = min delays):**

```text
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

**Recovery check (analogous to setup):**
   - Slack = Tperiod - Tc2q(reset_source) - Tcomb - Trecovery - Tuncertainty

**Removal check (analogous to hold):**
   - Slack = Tc2q_min(reset_source) + Tcomb_min - Tremoval - Tuncertainty_hold

Recovery/removal violations on reset are the root cause of many silicon bugs. This is why **asynchronous reset assertion / synchronous reset deassertion** is mandatory.

---

## 4. Reading a PrimeTime Timing Report

### 4.1 Annotated Setup Check Report

```text
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

```verilog
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

```tcl
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

```text
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

```text
                     CLK_ROOT
                        |
                    [BUF_common]  <- This buffer is shared
                     /        \
                [BUF_L]      [BUF_C]
                   |            |
               Launch_FF    Capture_FF
```

**Without CRPR (setup analysis):**

```text
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

```text
CRPR = BUF_common_max - BUF_common_min = 105 - 95 = 10ps

Adjusted capture clock = 180.5 + 10 = 190.5ps
Adjusted skew = 190.5 - 189 = +1.5ps  (now slightly helps setup)

CRPR recovered 10ps of pessimism!
```

### 6.3 Detailed CRPR Worked Example: Multi-Level Common Path

```ascii-graph
Consider a clock tree where the launch and capture paths share THREE
levels of buffers before diverging:

                           CLK_ROOT (t=0)
                               |
                          [BUF1] (100ps)
                               |
                          [BUF2] (80ps)
                               |
                          [BUF3] (60ps)
                          /          \
                    [BUF4_L]        [BUF4_C]
                    (50ps)          (70ps)
                       |                |
                  [BUF5_L]          [BUF5_C]
                  (40ps)            (55ps)
                       |                |
                   Launch_FF        Capture_FF

OCV derate: early = 0.93, late = 1.07 (7% derate)

Step 1: Identify common path (CLK_ROOT → BUF1 → BUF2 → BUF3)
  Common path has 3 buffers: BUF1, BUF2, BUF3
  Divergence point: output of BUF3

Step 2: Compute launch path (LATE derate = 1.07, worst for data arrival)
  BUF1: 100 * 1.07 = 107.0 ps
  BUF2:  80 * 1.07 =  85.6 ps
  BUF3:  60 * 1.07 =  64.2 ps    ← common path ends here
  BUF4_L: 50 * 1.07 = 53.5 ps
  BUF5_L: 40 * 1.07 = 42.8 ps
  Total launch clock = 107.0 + 85.6 + 64.2 + 53.5 + 42.8 = 353.1 ps

Step 3: Compute capture path (EARLY derate = 0.93, worst for data required)
  BUF1: 100 * 0.93 =  93.0 ps
  BUF2:  80 * 0.93 =  74.4 ps
  BUF3:  60 * 0.93 =  55.8 ps    ← common path ends here
  BUF4_C: 70 * 0.93 = 65.1 ps
  BUF5_C: 55 * 0.93 = 51.15 ps
  Total capture clock = 93.0 + 74.4 + 55.8 + 65.1 + 51.15 = 339.45 ps

Step 4: Compute pessimistic skew (WITHOUT CRPR)
  Skew = capture - launch = 339.45 - 353.1 = -13.65 ps (hurts setup)

Step 5: Compute CRPR credit
  Common path with LATE derate: 107.0 + 85.6 + 64.2 = 256.8 ps
  Common path with EARLY derate: 93.0 + 74.4 + 55.8 = 223.2 ps
  CRPR = 256.8 - 223.2 = 33.6 ps

Step 6: Apply CRPR
  Adjusted capture clock = 339.45 + 33.6 = 373.05 ps
  Adjusted skew = 373.05 - 353.1 = +19.95 ps (HELPS setup!)

  Without CRPR: skew = -13.65 ps (pessimistic, hurts setup by 13.65 ps)
  With CRPR:    skew = +19.95 ps (realistic, helps setup by 19.95 ps)
  Total recovered: 33.6 ps of pessimism

  For a 1 GHz clock with 50ps setup time and 0.10ns uncertainty:
    Without CRPR: available data time = 1000 - 50 - 100 - (-13.65) = 863.65 ps
    With CRPR:    available data time = 1000 - 50 - 100 + 19.95  = 869.95 ps
    Net gain:     6.3 ps additional slack

  For a tighter design (500 MHz, 200ps logic path):
    This 33.6 ps recovery could be the difference between pass and fail!
```

**CRPR Rules:**
- CRPR only applies to the COMMON portion of launch and capture clock paths
- The common path is identified by tracing both paths backward from the
  endpoints until they diverge
- CRPR credit = common_path_late - common_path_early
- For hold analysis: CRPR is applied similarly but with reversed derates
- CRPR is computed PER PATH PAIR (different path pairs may have different
  common path lengths and thus different CRPR credits)
- CRPR is automatically computed by PrimeTime when enabled -- no manual
  intervention needed

### 6.3 CRPR in PrimeTime

```text
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

```text
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

```text
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

```text
Path with 20 cells, each with nominal delay 50ps:
  Nominal path delay = 20 * 50ps = 1000ps

  With flat OCV (5%):
    Late path delay = 1000 * 1.05 = 1050ps  (50ps pessimism)

  With AOCV (depth=20, late derate=1.025):
    Late path delay = 1000 * 1.025 = 1025ps  (25ps pessimism)

  AOCV saved 25ps of pessimism!
```

### 7.2.1 OCV / AOCV / POCV Worked Comparison with Identical Path

```ascii-graph
Consider a specific timing path for setup analysis:
  15 cells in the data path, each with nominal delay = 40ps
  Clock period = 1.0 ns (1 GHz)
  T_c2q = 80ps, T_setup = 50ps, T_uncertainty = 80ps
  Clock skew = 0ps (ideal for simplicity)
  Data path nominal delay = 15 * 40 = 600ps
  Data arrival (nominal) = T_c2q + 600 = 80 + 600 = 680ps
  Data required = T_period - T_setup - T_uncertainty = 1000 - 50 - 80 = 870ps

  Nominal slack = 870 - 680 = 190ps (comfortable)

─── Method 1: Flat OCV (derate = 5%) ───

  Launch clock (early derate): 80 * 0.95 = 76ps
  Data path (late derate):     600 * 1.05 = 630ps
  Capture clock (late derate): assumed 0 for this example
  Setup time (late derate):    50 * 1.05 = 52.5ps
  Uncertainty: 80ps

  Data arrival = 76 + 630 = 706ps
  Data required = 1000 - 52.5 - 80 = 867.5ps
  OCV slack = 867.5 - 706 = 161.5ps

  Pessimism vs nominal: 190 - 161.5 = 28.5ps consumed by OCV

─── Method 2: AOCV (depth-dependent derate) ───

  Data path has depth=15 cells.
  From AOCV table (interpolate between depth 10 and 20):
    Late derate at depth 15 ≈ 1.030 (between 1.035 and 1.025)

  Launch clock (early, depth=1): 80 * 0.92 = 73.6ps
  Data path (late, depth=15):    600 * 1.030 = 618ps
  Setup time (late, depth=1):    50 * 1.08 = 54ps
  Uncertainty: 80ps

  Data arrival = 73.6 + 618 = 691.6ps
  Data required = 1000 - 54 - 80 = 866ps
  AOCV slack = 866 - 691.6 = 174.4ps

  AOCV recovered: 174.4 - 161.5 = 12.9ps vs flat OCV

─── Method 3: POCV (sigma = 3ps per cell) ───

  Each cell: mean = 40ps, sigma = 3ps (7.5% of mean)
  Data path mean  = 15 * 40 = 600ps
  Data path sigma = sqrt(15 * 3^2) = sqrt(135) = 11.62ps
  3-sigma data path delay = 600 + 3 * 11.62 = 634.86ps

  Clock path (1 cell): mean = 80ps, sigma = 3ps
  3-sigma early clock = 80 - 3 * 3 = 71ps (subtract for early)

  Setup time: mean = 50ps, sigma = 2ps
  3-sigma late setup = 50 + 3 * 2 = 56ps

  Data arrival = 71 + 634.86 = 705.86ps
  Data required = 1000 - 56 - 80 = 864ps
  POCV slack = 864 - 705.86 = 158.14ps

  Wait -- POCV looks worse than AOCV? Let's reconsider with proper derates:

  Actually, POCV sigma should be smaller. For a well-characterized 7nm library:
    sigma per cell ≈ 2% of mean = 0.8ps (not 3ps)
    Data path sigma = sqrt(15 * 0.64) = sqrt(9.6) = 3.10ps
    3-sigma data path = 600 + 3 * 3.10 = 609.3ps

    Clock sigma = 0.8ps
    3-sigma early clock = 80 - 3 * 0.8 = 77.6ps

    Setup sigma = 0.5ps
    3-sigma late setup = 50 + 3 * 0.5 = 51.5ps

    Data arrival = 77.6 + 609.3 = 686.9ps
    Data required = 1000 - 51.5 - 80 = 868.5ps
    POCV slack = 868.5 - 686.9 = 181.6ps

  Comparison:
    Flat OCV:   161.5 ps slack (28.5ps pessimism vs nominal)
    AOCV:       174.4 ps slack (15.6ps pessimism)
    POCV:       181.6 ps slack (8.4ps pessimism)

  POCV recovered: 181.6 - 161.5 = 20.1ps vs flat OCV
  POCV recovered: 181.6 - 174.4 =  7.2ps vs AOCV

  Key insight: POCV advantage grows with path depth.
  For depth=50 path (50 * 40ps = 2000ps):
    Flat OCV:  2000 * 1.05 = 2100ps (100ps pessimism)
    AOCV:      2000 * 1.015 = 2030ps (30ps pessimism)
    POCV:      2000 + 3 * sqrt(50 * 0.64) = 2000 + 17ps = 2017ps (17ps)
    POCV saves 83ps vs flat OCV, 13ps vs AOCV on deep paths!
```

### 7.3 POCV (Parametric OCV) / SOCV (Statistical OCV)

POCV models each cell's delay as a Gaussian random variable:

```text
Cell delay = mean + k * sigma

Where:
  mean  = nominal delay from .lib
  sigma = standard deviation (from POCV .lib extension)
  k     = number of sigma for coverage target
```

**Path delay statistics:**

```text
Path mean  = sum(mean_i)    for all cells i in the path
Path sigma = sqrt(sum(sigma_i^2))    (independent random variables)
```

Note: sigma adds in **quadrature** (root-sum-of-squares), NOT linearly. This is the key insight -- for a path with N identical cells:

```text
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

```ascii-graph
          Wire A (aggressor)
  ═══════════════════════════
          Cc (coupling cap)
  ───────────────────────────
          Wire B (victim)
```

The victim wire sees an effective capacitance that depends on the aggressor's switching:

```text
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

- **Delta delay** = `delay_with_crosstalk - delay_without_crosstalk`

For a victim net with coupling cap Cc to an aggressor:
Worst case delta = Cc * (V_aggressor / V_victim) * R_victim

**Simplified Miller factor approach:**
   - Same-direction switching:  effective Cc = Cc * (1 - slew_ratio)
   - Opposite-direction switching: effective Cc = Cc * (1 + slew_ratio)

where slew_ratio approximates the temporal overlap

**Numerical example:**

Victim wire: R = 50 ohm, C_ground = 20 fF
Coupling cap to aggressor: Cc = 15 fF
Aggressor switches in opposite direction (harmful)

**Without crosstalk:**
   - delay ~ 0.69 * R * C_total = 0.69 * 50 * 35fF = 1.21 ps

- **With crosstalk (Miller factor** = `2x on Cc):`
   - C_effective = 20 + 2*15 = 50 fF
   - delay ~ 0.69 * 50 * 50fF = 1.73 ps

- **Delta delay** = `0.52 ps  (43% increase!)`

At advanced nodes with tight pitch (7nm: ~28nm pitch), coupling capacitance can be 50-70% of total wire capacitance, making SI-aware STA critical.

### 8.3 Glitch Analysis

A glitch occurs when the aggressor switches but the victim is quiet. The coupling injects a voltage bump on the victim.

```text
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

```text
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

```text
Before useful skew:
  Path A->B: slack = -50ps (FAIL)
  Path C->B: slack = +200ps (comfortable)

CTS intentionally delays B's clock by 50ps:
  Path A->B: slack = -50 + 50 = 0ps (MET)
  Path C->B: slack = +200 - 50 = +150ps (still fine)
```

The CTS tool solves a global optimization problem: minimize total negative slack across all paths by redistributing skew within the clock tree. Constraint: no path may go negative after skew redistribution.

### 11.5 Post-CTS Timing Closure

```text
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

```ascii-graph
Typical PVT corners for a 7nm design:

Corner           Process   Voltage   Temp     Parasitics   Primary Check
────────────────────────────────────────────────────────────────────────
SS_0p72V_125C    Slow-Slow  0.72V    125°C    Cmax/RCmax   Setup (hot)
SS_0p72V_m40C    Slow-Slow  0.72V    -40°C    Cmax/RCmax   Setup (cold)*
FF_0p88V_m40C    Fast-Fast  0.88V    -40°C    Cmin/RCmin   Hold (cold)
FF_0p88V_125C    Fast-Fast  0.88V    125°C    Cmin/RCmin   Hold (hot)*
TT_0p80V_25C     Typical    0.80V    25°C     Ctyp         Nominal/Power

* Temperature inversion corners: at advanced nodes (< 28nm), cold + slow
  voltage can make cells slower due to Vth increase at low temperature.
  This is why SS_cold must be checked for setup in addition to SS_hot.
  Similarly, FF_hot must be checked for hold because hot + fast voltage
  is no longer automatically the worst hold corner.

Which corner is worst for each check:
  ────────────────────────────────────
  Setup (data arrives too late):
    Worst = slow process + low voltage + high temp (traditional)
    But at advanced nodes, also check: slow + low voltage + LOW temp
    Reason: Vth increases at cold → at low Vdd near Vth,
            increased Vth can dominate mobility improvement → slower cells

  Hold (data arrives too early):
    Worst = fast process + high voltage + low temp (traditional)
    But at advanced nodes, also check: fast + high voltage + HIGH temp
    Reason: at high temp, lower Vth means even faster cells;
            combined with high voltage, hold may be worse at hot

  Why both BC-WC and MCMM are needed:
  ────────────────────────────────────
  BC-WC (Best-Case / Worst-Case): simple 2-corner analysis
    SS corner for setup, FF corner for hold
    Fast, but misses temperature inversion effects

  MCMM: exhaustive multi-corner multi-mode analysis
    All combinations of corners x modes
    Catches temperature inversion, voltage droop scenarios, mode-specific issues
    Industry standard for signoff at 7nm and below

  Typical signoff corner combinations (minimum for tapeout):
    Setup:  SS_0p72V_125C + SS_0p72V_m40C  (2 corners)
    Hold:   FF_0p88V_m40C + FF_0p88V_125C  (2 corners)
    DRC:    All corners (max_transition, max_capacitance)
    Power:  TT_0p80V_25C with real activity (SAIF)

  With extraction corners:
    Setup:  SS + Cmax (worst capacitance → more delay)
    Hold:   FF + Cmin (best capacitance → less delay)
    For wire-dominated paths: also check SS + RCmax (worst R AND C)

  Total signoff scenarios for a complex SoC:
    4 timing corners x 3 extraction corners x 4 modes = 48 scenarios
    Each needs: .lib + SPEF + SDC + netlist
    Runtime: 48 STA runs, each 2-8 hours = 4-16 days on a single machine
    Parallelized across a compute farm: typically 1-2 day turnaround

Modes (functional scenarios):
  - func_mode: normal operation, highest frequency
  - scan_mode: test mode, slower clock, shift/capture constraints
  - sleep_mode: low power, most clocks gated, leakage focus

MCMM matrix: every corner × every mode = a "scenario"
Each scenario has its own SDC (constraints), SPEF (parasitics), library

Typical signoff: 10-30 scenarios for a complex SoC
```

### 12.2 Signoff Criteria

```text
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

## 13. .lib Format Deep-Dive

### 13.1 Library Structure

```text
A .lib (Liberty) file is the ASCII representation of a standard cell timing library.
It is the primary interface between the foundry cell library and EDA tools (STA, PnR).

High-level structure:

library (my_library_n5) {
  /* Library-level attributes */
  delay_model : table_lookup;          /* NLDM */
  /* or delay_model : ccs; */          /* CCS  */
  time_unit : "1ns";
  voltage_unit : "1V";
  current_unit : "1mA";
  capacitive_load_unit (1, pf);        /* 1 pF */
  pulling_resistance_unit : "1kohm";
  operating_conditions (ss_0p72v_125c) {
    process : 1.0;
    temperature : 125.0;
    voltage : 0.72;
  }
  default_operating_conditions : ss_0p72v_125c;

  /* Lookup table templates (shared by all cells) */
  lu_table_template (delay_template_7x7) {
    variable_1 : input_net_transition;
    variable_2 : total_output_net_cap;
    index_1 ("0.01, 0.05, 0.1, 0.2, 0.5, 1.0, 2.0");
    index_2 ("0.005, 0.01, 0.02, 0.05, 0.1, 0.2, 0.5");
  }

  /* Cell definitions (500-2000 cells in a modern library) */
  cell (INVX1) { ... }
  cell (INVX2) { ... }
  cell (NAND2X1) { ... }
  cell (DFFRHQX1) { ... }
  ...
}

A modern standard cell library (e.g., TSMC N5):
  ~500-2000 cells total
  Categories: inverters, buffers, NAND/NOR, AOI/OAI, MUX, FF/latch,
              clock gating (ICG), fillers, tap cells, antenna diodes
  Drive strength variants: X1, X2, X4, X8, X16 (5-6 sizes per function)
  Vt variants: ULVT, LVT, SVT (RVT), HVT, ULHVT (3-5 thresholds per size)
  Total library permutations: function × drive × Vt = easily 500-2000 unique cells
```

### 13.2 Cell Definition and Pin Descriptions

```text
A typical combinational cell in .lib:

cell (NAND2X1) {
  area : 0.8;               /* Cell area in um² */
  cell_footprint : "nand2";

  /* Leakage power (per state) */
  cell_leakage_power : 2.5;                       /* average */
  leakage_power () { when : "!A & !B"; value : 1.2; }
  leakage_power () { when : "!A & B";  value : 3.8; }
  leakage_power () { when : "A & !B";  value : 3.6; }
  leakage_power () { when : "A & B";   value : 1.4; }

  /* Pin definitions */
  pin (A) {
    direction : input;
    capacitance : 0.0012;          /* Input pin cap (pF) */
    rise_capacitance : 0.0011;
    fall_capacitance : 0.0013;

    /* Internal power (energy consumed per transition, no load) */
    internal_power () {
      related_pin : "A";
      rise_power (energy_template_7x1) { values("..."); }
      fall_power (energy_template_7x1) { values("..."); }
    }
  }

  pin (B) {
    direction : input;
    capacitance : 0.0014;
    ...
  }

  pin (Y) {
    direction : output;
    max_capacitance : 0.15;       /* Design rule: max load this pin can drive */
    min_capacitance : 0.0;

    function : "!(A & B)";        /* Boolean function */

    /* Timing arcs: one per related_pin */
    timing () {
      related_pin : "A";
      timing_sense : negative_unate;
      timing_type : combinational;

      cell_rise (delay_template_7x7) {
        index_1 ("0.01, 0.05, ...");   /* input slew */
        index_2 ("0.005, 0.01, ...");  /* output load */
        values ( ... );
      }
      cell_fall (delay_template_7x7) { values ( ... ); }
      rise_transition (delay_template_7x7) { values ( ... ); }
      fall_transition (delay_template_7x7) { values ( ... ); }
    }

    timing () {
      related_pin : "B";
      timing_sense : negative_unate;
      timing_type : combinational;
      cell_rise (...) { ... };
      cell_fall (...) { ... };
      rise_transition (...) { ... };
      fall_transition (...) { ... };
    }
  }
}
```

### 13.3 Indexed Lookup Tables and Interpolation

```ascii-graph
Each timing arc stores delay/slew as a 2D table indexed by:
  index_1 = input transition time (slew at the input pin)
  index_2 = output load capacitance (total cap seen at the output pin)

The STA tool performs bilinear interpolation:

  Given input slew S and output load C:

  1. Find bounding indices: S_lo, S_hi, C_lo, C_hi from table indices
  2. Retrieve four table values:
     V00 = table[S_lo][C_lo]   V01 = table[S_lo][C_hi]
     V10 = table[S_hi][C_lo]   V11 = table[S_hi][C_hi]
  3. Interpolate in slew dimension:
     V_lo = V00 + (C - C_lo)/(C_hi - C_lo) * (V01 - V00)
     V_hi = V10 + (C - C_lo)/(C_hi - C_lo) * (V11 - V10)
  4. Interpolate in load dimension:
     Result = V_lo + (S - S_lo)/(S_hi - S_lo) * (V_hi - V_lo)

If the input slew or load falls outside the table range, the tool extrapolates
(uses the two nearest points to extend the table). This is a source of inaccuracy
and tools warn about out-of-range table lookups.

Table sizes:
  NLDM: typically 7x7 or 10x10 (49 or 100 entries per timing arc)
  CCS:  4D tables (slew × load × time × voltage) → much larger
```

### 13.4 NLDM vs CCS vs ECSM Timing Models

```text
NLDM (Non-Linear Delay Model):
  - Stores: cell delay (ns) and output slew (ns) as 2D lookup tables
  - Model: voltage source with fixed rise/fall time driving lumped C load
  - Accuracy: ±10-15% at 28nm and above
  - Library size: ~20-100 MB per corner
  - Adequate for: 180nm to 28nm

CCS (Composite Current Source, Synopsys):
  - Stores: output current waveform I(t) instead of delay
  - Model: time-varying current source driving RC interconnect
  - Also stores: receiver pin capacitance as function of input transition
  - Captures: Miller effect, multi-stage driver turn-on, waveform shape
  - Accuracy: ±2-3% at 7nm and below
  - Library size: ~200-500 MB per corner
  - Required for: 16nm and below (FinFET)
  - Runtime: 2-4× slower than NLDM

ECSM (Effective Current Source Model, Cadence):
  - Cadence's equivalent to CCS
  - Stores: voltage waveform at output pin (V(t)) instead of current
  - Equivalent accuracy to CCS (both model waveform shape)
  - Some tools support both; some are vendor-specific

When to use which:
  | Node   | Recommended Model |
  |--------|-------------------|
  | >= 28nm | NLDM (sufficient) |
  | 16nm   | CCS or ECSM       |
  | 7nm    | CCS mandatory     |
  | 5nm    | CCS mandatory     |
  | 3nm    | CCS + advanced SI |

Multi-corner libraries:
  Each PVT corner has its own .lib:
    ss_0p72v_125c.lib    (slow, setup signoff)
    ff_0p88v_m40c.lib    (fast, hold signoff)
    tt_0p80v_25c.lib     (typical, power estimation)
  Total library data for one design: 10-50 GB across all corners
```

### 13.5 State-Dependent Timing (when Condition)

```text
Some cells have different timing depending on the logic state of other pins.
This is modeled with the "when" condition in the timing arc.

Example: A 2:1 MUX (I0, I1 select inputs, S = select)

  timing () {
    related_pin : "S";
    when : "!I0 & !I1";         /* Arc active only when both inputs are 0 */
    timing_sense : non_unate;
    cell_rise (...) { ... };
    cell_fall (...) { ... };
  }

  timing () {
    related_pin : "S";
    when : "I0 & I1";           /* Different timing when both inputs are 1 */
    timing_sense : non_unate;
    cell_rise (...) { ... };
    cell_fall (...) { ... };
  }

The STA tool evaluates ALL "when" conditions and takes the worst-case across
valid states. For a MUX with N control combinations, there can be N timing arcs
per related_pin, each with potentially different delay values.

This is also used for:
  - Tri-state buffers (enable-dependent timing)
  - Adders (carry-dependent propagation delay)
  - Clock gating cells (clock + enable state-dependent behavior)

Sequential cells also use state-dependent setup/hold:

  timing () {
    related_pin : "D";
    timing_type : setup_rising;
    when : "!RN";               /* Setup only when reset is deasserted */
    rise_constraint (...) { ... };
    fall_constraint (...) { ... };
  }
```

---

## 14. SPEF Walkthrough

### 14.1 SPEF Format Header

```text
SPEF (Standard Parasitic Exchange Format, IEEE 1481-1999):
  The interchange format for extracted parasitic RC data from layout to STA tools.

*SPEF "IEEE 1481-1999"
*DESIGN "video_proc_top"
*DATE "2024-03-15 14:30:00"
*VENDOR "Synopsys StarRC"
*PROGRAM "StarRC"
*VERSION "v2023.12"
*DESIGN_FLOW "PIN_CAP NONE" "NAME_SCOPE LOCAL"
*DIVIDER /
*DELIMITER :
*BUS_DELIMITER [ ]

Units section (must match .lib time/voltage/current units):
*T_UNIT 1 NS                /* Time in nanoseconds */
*C_UNIT 1 PF                /* Capacitance in picofarads */
*R_UNIT 1 OHM               /* Resistance in ohms */
*L_UNIT 1 HENRY             /* Inductance in henries (rarely used) */
```

### 14.2 Name Map Section

```text
*NAME_MAP
*1 u_core/u_alu/g127/Y
*2 u_core/u_alu/g128/A
*3 u_core/u_alu/net_n1
*4 u_core/wb_reg[3]/D
*5 u_core/u_alu/g127/Z
...

Name map assigns integer IDs to hierarchical net/instance names.
Reduces file size dramatically (names are repeated many times in
large designs with millions of nets).

After name map, all references use *N instead of the full path:
  *CONN
  *I *5 O     (instead of *I u_core/u_alu/g127/Z O)
```

### 14.3 Port Section

```text
*PORTS
*1 u_core/clk_in I *C 0.0023      /* Input port, cap = 2.3 fF */
*2 u_core/data_out[0] O *C 0.0015  /* Output port, cap = 1.5 fF */
*3 u_core/vdd P *C 0.0             /* Power port, no cap */
*4 u_core/gnd G *C 0.0             /* Ground port, no cap */

Port direction codes:
  I = input
  O = output
  B = bidirectional
  P = power
  G = ground
  X = don't care

The *C value is the port's intrinsic capacitance (from the .lib pin cap).
```

### 14.4 Net Section (RC Parasitics)

```ascii-graph
*D_NET *3 0.0523                   /* Net name_map_ref, total_cap = 52.3 fF */

*CONN                              /* Connections to this net */
*I *5 O                            /* Instance pin: g127/Z (output/driver) */
*I *2 I *C 0.0087                  /* Instance pin: g128/A (input), pin cap = 8.7 fF */
*I *4 I *C 0.0078                  /* Instance pin: wb_reg[3]/D (input), pin cap = 7.8 fF */

*CAP                               /* Capacitance entries */
*1 *5 0.0102                       /* Cap at driver pin: 10.2 fF */
*2 *2 0.0087                       /* Cap at receiver pin g128/A: 8.7 fF (from .lib) */
*3 *3 0.0156                       /* Cap at internal node n1: 15.6 fF (wire cap) */
*4 *4 0.0078                       /* Cap at receiver pin wb_reg[3]/D: 7.8 fF */
*5 *6 0.0100                       /* Cap at internal node n2: 10.0 fF */

*RES                               /* Resistance entries */
*1 *5 *3 2.45                      /* R from driver pin to n1: 2.45 ohm */
*2 *3 *2 1.83                      /* R from n1 to g128/A: 1.83 ohm */
*3 *3 *6 3.12                      /* R from n1 to n2: 3.12 ohm */
*4 *6 *4 2.67                      /* R from n2 to wb_reg[3]/D: 2.67 ohm */

*END                               /* End of this net */

RC network interpretation:

          2.45Ω       1.83Ω
  g127/Z ---[R1]--- n1 ---[R2]--- g128/A
    |           |                  |
  10.2fF     15.6fF             8.7fF
               |
             [R3] 3.12Ω
               |
              n2
               |
             [R4] 2.67Ω
               |
          wb_reg/D
               |
             7.8fF + 10.0fF

Total net capacitance = 10.2 + 8.7 + 15.6 + 7.8 + 10.0 = 52.3 fF (matches *D_NET)
```

### 14.5 RC Parasitics and the Elmore Delay Model

```verilog
Elmore delay: sum of (each resistance × downstream capacitance)

For the above RC tree, delay from g127/Z to g128/A:

  T_Elmore = R1 × (C_n1_downstream) + R2 × C_g128
           = 2.45 × (15.6 + 10.0 + 8.7 + 7.8) + 1.83 × 8.7
           = 2.45 × 42.1fF + 1.83 × 8.7fF
           = 103.1 + 15.9
           = 119.0 fF·Ω = 0.119 ps (units: fF × Ω = 10⁻¹⁵ × 10⁰ = 10⁻¹⁵ s = 1 fs)

  Wait — correct units: fF × Ω = 10⁻¹⁵ F × 10⁰ Ω = 10⁻¹⁵ s = 1 fs
  So 119 fF·Ω = 0.119 ps... this is tiny for an on-chip net.

  For a longer net (e.g., 1mm wire at 0.2Ω/μm, 0.2fF/μm):
  R_total = 200 Ω, C_total = 200 fF
  T_Elmore ≈ 0.38 × R × C = 0.38 × 200 × 200fF = 15.2 ps

Elmore is a first-order upper bound. It overestimates delay because it assumes
all downstream capacitance is charged through the full resistance path.

More accurate models used in signoff STA:
  - Arnoldi / AWE (Asymptotic Waveform Evaluation): models multiple poles
  - PRIMA: passive reduced-order interconnect macromodeling
  - These capture the non-exponential waveform shape more accurately
```

### 14.6 Distributed vs Lumped RC

**Lumped RC model:**
   - Entire wire modeled as a single R and single C.
   - Delay = R × C
   - Accurate only for very short wires (< 50 μm at 7nm).

**Distributed RC model (Pi, T, or ladder):**
   - Wire broken into multiple RC segments.
   - For N segments: T_Elmore = R × C / (2N) → as N→∞: T = R × C / 2

Lumped:     T = R × C
Distributed: T = 0.38 × R × C (Elmore for uniform RC line, factor = 0.38)

The 0.38 factor comes from the integral:
T_Elmore = ∫₀ᴸ R(x) × C(x,L) dx = RC × L² / 2
Normalized to R×C = RC × L², so factor = 0.5 for one segment, 0.38 for continuous.

SPEF represents the distributed RC explicitly (multiple R and C segments).
The extraction tool (StarRC, Calibre xRC, QRC) determines how many segments
to use based on wire length, frequency, and accuracy requirements.

For signoff STA: detailed SPEF with coupled RC (includes Cc between adjacent nets).
For pre-route STA: estimated SPEF from wireload models or virtual route estimation.

---

## 15. Clock Gating Checks

### 15.1 What Are Clock Gating Checks

```ascii-graph
Clock gating is used extensively to reduce dynamic power by disabling the clock
to idle modules. The gating element is typically an Integrated Clock Gating (ICG)
cell: a latch + AND gate combination.

Standard ICG cell (e.g., TLATNTSCAX4):
                     ┌───────┐
  CLK ──────────────┤       │
                     │  AND  ├─── GATED_CLK
  EN ──┬──────┐     │       │
       │LATCH │─────┤       │
       └──────┘     └───────┘
  (latch transparent when CLK = low)

The latch captures EN while CLK is low and holds it stable when CLK goes high.
This prevents glitches on the gated clock output.

Clock gating checks ensure the enable signal (EN) is stable during the
clock's active transition. If EN changes while CLK is transitioning:
  - A partial pulse (glitch) appears on GATED_CLK
  - Downstream FFs see a narrow clock pulse
  - Potential metastability → data corruption → system failure

These are formal timing checks in STA, similar to setup/hold on data FFs.
```

### 15.2 Clock Gating Setup Check

```verilog
Setup check: EN must be stable BEFORE the clock rising edge (for positive-edge ICG).

  The latch must capture the final EN value before CLK goes high (latch closes).

  CLK:  ____/‾‾‾‾‾‾‾‾\____
  EN:  ------\_________/---    ← EN must transition here at latest

  Setup requirement:
    EN_launch_time + EN_path_delay + ICG_setup < CLK_rising_edge

  In .lib, the ICG cell has a setup arc:
    timing () {
      related_pin : "CLK";
      timing_type : setup_rising;
      rise_constraint (setup_template) { ... }
      fall_constraint (setup_template) { ... }
    }

  Common setup violation: EN arrives too late relative to clock.
  The enable logic (often a state machine output) has too many levels of logic
  between the enable register and the ICG cell.

  Fix: pipeline the enable signal, reduce logic depth, or use a faster enable path.
```

### 15.3 Clock Gating Hold Check

```text
Hold check: EN must remain stable AFTER the clock rising edge.

  After CLK goes high, the latch output must not change. The latch hold time
  ensures EN does not glitch during the clock transition window.

  CLK:  ____/‾‾‾‾‾‾‾‾\____
  EN:  ----------------\___  ← EN must remain stable past this point

  Hold requirement:
    EN_launch_time + EN_path_delay_min > CLK_rising_edge + ICG_hold

  In .lib:
    timing () {
      related_pin : "CLK";
      timing_type : hold_rising;
      rise_constraint (hold_template) { ... }
      fall_constraint (hold_template) { ... }
    }

  Common hold violation: EN glitches due to crosstalk or short path from enable FF.
  Because the enable path is often short (1-2 gates), hold violations are common
  when the ICG and enable FF are physically close.

  Fix: add delay buffers on the enable path, increase wire length during routing.
```

### 15.4 How STA Tools Model Clock Gating Checks

```text
The ICG cell in .lib contains specific timing arcs that trigger clock gating checks:

cell (TLATNTSCAX4) {
  /* Latch: EN sampled on falling CLK, held through rising CLK */
  pin (EN) {
    direction : input;
    timing () {
      related_pin : "CLK";
      timing_type : setup_rising;    /* Triggers gating setup check */
      rise_constraint (...) { ... }
      fall_constraint (...) { ... }
    }
    timing () {
      related_pin : "CLK";
      timing_type : hold_rising;     /* Triggers gating hold check */
      rise_constraint (...) { ... }
      fall_constraint (...) { ... }
    }
  }
  pin (CLK) { direction : input; }
  pin (GATED_CLK) {
    direction : output;
    /* Clock propagation: CLK → GATED_CLK with insertion delay */
    timing () {
      related_pin : "CLK";
      timing_type : rising_edge;
      cell_rise (...) { ... }
      cell_fall (...) { ... }
    }
  }
}

In PrimeTime reports, clock gating checks appear as:

  Startpoint: u_core/enable_reg/Q  (rising edge-triggered FF clocked by sys_clk)
  Endpoint:   u_core/u_icg/EN      (clock gating check for rising edge of sys_clk)
  Check type: clock gating setup

  The tool treats this identically to a FF setup/hold check but with the ICG's
  constraint values instead of a data FF's setup/hold values.
```

### 15.5 Why Clock Gating Checks Matter

```tcl
Glitch on gated clock → metastability → data corruption:

  If EN changes during CLK rise:
  1. AND gate output produces a narrow pulse (glitch) instead of a full clock edge
  2. Downstream FF sees a clock pulse narrower than its minimum pulse width
  3. FF's master latch may partially capture data → metastable output
  4. Metastable value propagates → corrupted state in pipeline or FSM
  5. System-level failure: wrong data written, FSM stuck in illegal state

  This is one of the most insidious bugs in silicon:
    - Rarely triggered (depends on specific enable timing)
    - Not caught by functional simulation (no timing in RTL sim)
    - Only visible in STA or on silicon with unlucky process corner

  Real-world example:
    A mobile SoC had intermittent crashes in the display controller.
    Root cause: clock gating enable failed hold by 3ps at FF corner.
    Fix: one buffer added to enable path. Cost: $0. One silicon re-spin: $5M+.

Best practices:
  1. Always use ICG cells (never hand-instantiated latch + AND)
  2. Register enable signals as close to the ICG as possible
  3. Apply max_transition constraint on enable pins (slew affects ICG timing)
  4. Run clock gating checks at ALL MCMM corners (hold is critical at FF corner)
  5. In SDC: ensure clock uncertainty covers enable-to-clock relationship
  6. Use `report_timing -check_type clock_gating` to specifically audit these checks
```

---

## 16. GAA (Gate-All-Around) Variation Analysis

### 17.1 Nanosheet Width Variation

At N2 and beyond, Gate-All-Around (GAA / nanosheet) transistors introduce new variation sources beyond what FinFETs experienced:

```ascii-graph
GAA nanosheet structure:
  ┌────────────────────────────────┐
  │         Gate Metal             │
  │  ┌──────────────────────────┐  │
  │  │   NS1 (nanosheet 1)      │  │
  │  ├──────────────────────────┤  │
  │  │   NS2 (nanosheet 2)      │  │
  │  ├──────────────────────────┤  │
  │  │   NS3 (nanosheet 3)      │  │
  │  └──────────────────────────┘  │
  │         Gate Metal             │
  └────────────────────────────────┘

New variation sources at N2:
  1. Nanosheet width variation:
     - Sheet width W_ns defines drive strength
     - Variation: ±1-3 nm per sheet (lithography + etch)
     - Wider sheets = more drive, but also more capacitance
     - Device matching: adjacent sheets match well; distant sheets less so

  2. Nanosheet thickness variation:
     - Internal oxidation defines sheet gap (2-5 nm)
     - Thickness variation affects Vth and mobility
     - More critical than FinFET fin height variation

  3. Work function variation (WFV):
     - Metal gate granularity causes random Vth shifts
     - Different metal crystal orientations → different work functions
     - Per-device variation: ~5-15 mV sigma at N2
     - This is ADDITIVE to all other variation sources

  4. Contact resistance variation:
     - Source/drain contact resistance: 50-200 Ω·μm at N2
     - Variation due to silicide formation non-uniformity
     - Impacts access time and drive current
```

### 17.2 POCV / LOCV at 2nm Nodes

```ascii-graph
POCV at N2: Increased variability requires more sophisticated modeling

  Variation budget breakdown at N2 (VDD = 0.55V):
  ┌───────────────────────────────┬───────────────────┐
  │ Source                         │ 3-sigma impact    │
  ├───────────────────────────────┼───────────────────┤
  │ Global process (die-to-die)   │ ±8-12%            │
  │ Local Vth variation (WFV)     │ ±3-5% per device  │
  │ Nanosheet width               │ ±2-4% per device  │
  │ Voltage (IR drop)             │ ±5-8%             │
  │ Temperature gradient          │ ±3-5%             │
  │ Aging (NBTI/ HCI, 3yr)       │ ±5-10%            │
  └───────────────────────────────┴───────────────────┘

  Total variation at N2 can be 20-30% without POCV averaging

  POCV sigma values (typical N2, 7.5T library):
    Inverter (X1):  mean = 8 ps,  sigma = 1.5 ps
    Inverter (X8):  mean = 5 ps,  sigma = 1.0 ps
    NAND2 (X2):     mean = 12 ps, sigma = 2.0 ps
    DFF (Q-to-Q):   mean = 25 ps, sigma = 4.0 ps

  LOCV (Location-based OCV) — emerging for N2:
    - Combines POCV with physical distance derating
    - sigma increases with distance between cells
    - sigma_eff = sigma_base + k_distance * distance
    - Better accuracy for large dies (>500 mm²) where
      spatial correlation matters

  Impact on signoff:
    - More extraction corners needed (6-8 vs 4 at N5)
    - POCV tables include WFV contribution
    - Aging derating must be included in signoff corners
    - Self-heating variation adds another dimension
```

### 17.3 AI Accelerator STA

```tcl
Multi-GPU die STA (e.g., B200 with dual-reticle GPU):

  Clock domains in a B200-class accelerator:
    - GPU core clock: ~1.8-2.2 GHz
    - HBM interface clock: ~3.2 GHz (HBM3E data rate 9.6 Gbps)
    - NVLink clock: ~2.0 GHz
    - PCIe clock: 250 MHz (reference)
    - NoC (Network-on-Chip) clock: ~1.5 GHz

  Die-to-die link timing:
    - UCIe standard package: 32 GT/s per lane
    - Forwarded clock architecture: source-synchronous
    - TX jitter budget: ~10-20 ps
    - RX setup/hold at die edge: must include:
      • Interposer wire delay (~50-200 ps depending on length)
      • μbump parasitic (~5-10 ps)
      • Clock-data compensation across PVT

    STA modeling for die-to-die:
      set_input_delay  -clock rx_clk -max 0.200 [get_ports rx_data*]
      set_input_delay  -clock rx_clk -min 0.050 [get_ports rx_data*]
      set_output_delay -clock tx_clk -max 0.200 [get_ports tx_data*]
      set_output_delay -clock tx_clk -min 0.050 [get_ports tx_data*]
      set_clock_uncertainty -setup 0.050 [get_clocks rx_clk]
      set_clock_uncertainty -hold  0.020 [get_clocks rx_clk]

  NVLink compliance timing:
    - 200+ GB/s aggregate bandwidth
    - Per-lane: 100 Gbps PAM-4 (NRZ equivalent)
    - TX eye margin: ~20-30 ps at target BER 1e-15
    - Must verify with IBIS-AMI models in STA extraction
```

### 17.4 MCMM for Advanced Packaging

```ascii-graph
Multi-chiplet signoff complexity:

  For a chiplet-based AI accelerator (e.g., AMD MI300X):
    - 8 compute chiplets + 1 I/O chiplet + HBM stacks
    - Each chiplet has its own PVT corner
    - Inter-chiplet links add inter-die corners
    - Package-level thermal variation: chiplets at different T

  MCMM scenarios for chiplet designs:
  ┌─────────────┬──────────┬──────────┬───────────┬───────────┐
  │ Scenario     │ Chiplet A│ Chiplet B│ HBM Stack │ Package T │
  ├─────────────┼──────────┼──────────┼───────────┼───────────┤
  │ SS_A, SS_B  │ SS, 0.7V │ SS, 0.7V │ SS, 0.9V  │ 105°C     │
  │ FF_A, SS_B  │ FF, 0.8V │ SS, 0.7V │ TT, 0.95V │ 85°C      │
  │ SS_A, FF_B  │ SS, 0.7V │ FF, 0.8V │ TT, 0.95V │ 85°C      │
  │ FF_A, FF_B  │ FF, 0.8V │ FF, 0.8V │ FF, 1.0V  │ -40°C     │
  └─────────────┴──────────┴──────────┴───────────┴───────────┘

  Additional extraction corners for inter-chiplet:
    - Interposer RDL: Cworst, RCworst, Cbest, RCbest
    - μbump parasitic: min/typical/max
    - TSV delay: min/max (if 3D stacking)

  Total scenario count:
    4 intra-die corners × 4 inter-die corners × 3 modes
    = 48+ scenarios per chiplet pair
    Signoff requires hierarchical STA with distributed analysis
```

### 17.5 Incremental STA for ECO Flows

```verilog
Incremental STA: Re-analyze only affected timing paths after ECO

  When to use:
    - After timing ECO (cell swap, buffer insertion)
    - After functional ECO (logic change in small region)
    - After metal fill insertion
    - After DRC fix (via optimization, wire widening)

  Flow:
    1. Run full-chip STA (baseline)
    2. Apply ECO changes
    3. Re-extract parasitics for affected region + margin
       (typically: changed cells + 50-100 μm boundary)
    4. Run incremental STA:
       update_timing -full -incremental
       (PrimeTime) or equivalent
    5. Report only changed paths + paths with delta > threshold

  Performance comparison:
    Full-chip STA:    8-24 hours (for 100M+ gate design)
    Incremental STA:  10-60 minutes (depends on ECO scope)
    Speedup:          10-100× for localized ECOs

  Correctness checks:
    - All paths through modified cells are re-analyzed
    - Paths with shared parasitic nodes are re-analyzed
    - Hold/setup interaction: verify ECO doesn't flip one to help other
    - Full STA recommended before final tapeout signoff
```

---

## Numbers to Memorize -- STA Quick Reference

| Quantity | Value | Why it matters |
|----------|-------|----------------|
| Typical FO4 delay at N5 | ~12-15 ps | Fundamental gate delay yardstick; used to estimate cycle-time budget |
| Setup time (typical DFF, N5) | 30-60 ps | Data must be stable before clock edge; directly consumes cycle time |
| Hold time (typical DFF, N5) | 20-40 ps | Data must remain stable after clock edge; negative hold time is a gift from library design |
| Clock uncertainty (post-CTS) | +/-20-50 ps | Accounts for jitter + residual skew; tightens both setup and hold margins |
| OCV derate (typical) | +/-5-10% | Flat derate applied to model within-die variation; overly pessimistic for deep paths |
| AOCV derate range | +/-3-8% (depth/distance-based) | Less pessimistic than flat OCV because variations average over many stages |
| POCV sigma (typical) | 2-5% per cell | Statistical model; sigma adds in quadrature, so deep paths benefit most |
| IR drop budget | <=5% VDD | Exceeding this slows cells enough to cause timing failures |
| Typical metal RC (M4, N5) | ~0.2 ohm/um, ~0.2 fF/um | Local interconnect; dominates short-wire delay at advanced nodes |
| Typical metal RC (M8, N5) | ~0.05 ohm/um, ~0.3 fF/um | Semi-global interconnect; used for clock and long data routes |
| Clock tree target skew | <100 ps (moderate designs) | Skew directly erodes timing margin; tighter for high-frequency designs |
| Clock tree target latency | 0.5-2 ns depending on size | High latency increases susceptibility to jitter and OCV |
| Setup slack target | >=0 (positive) | Negative setup slack means the path cannot meet frequency |
| Hold slack target | >=0 (positive, fixed with filler cells/buffering) | Hold failures are silicon killers; they cannot be fixed by slowing the clock |
| CRPR credit (typical) | 20-40% of clock uncertainty | Recovered pessimism from shared clock path; larger for balanced trees |
| Typical derate table entries | early/late, data/clock | Four entries cover setup/hold x launch/capture combinations |
| Multi-corner signoff corners | 3-5 (SS/TT/FF x low/high temp x low/high voltage) | Must cover temperature inversion and process extremes |
| LIB file cell count | ~500-2000 cells per library | Larger libraries give the optimizer more freedom but increase runtime |
| Typical design frequency | 500 MHz - 3 GHz (application-dependent) | Sets cycle time; determines how much slack budget is available |
