# Signal Integrity and Reliability — Senior Engineer Deep Dive

## Table of Contents
1. Crosstalk Fundamentals
2. Crosstalk Noise Analysis
3. Crosstalk Delay Impact
4. Electromigration (EM)
5. IR Drop Analysis
6. Power Grid Design
7. Thermal Analysis
8. Reliability Signoff Methodology

---

## 1. Crosstalk Fundamentals

### 1.1 Coupling Capacitance Model

```ascii-graph
Two adjacent parallel wires in an IC:

    GND plane (above)
  ═══════════════════════
      ↕ Ctop          ↕ Ctop
   ┌──────┐  Cc    ┌──────┐
   │Victim│←─┤├──→│Aggr. │    Cc = coupling capacitance
   └──────┘        └──────┘
      ↕ Cbot          ↕ Cbot
  ═══════════════════════
    GND plane (below)

  Ctotal_victim = Cground + Cc
  Cground = Ctop + Cbottom (capacitance to reference planes)

  Coupling ratio: Cc / (Cc + Cground)
    At 7nm, lower metals: Cc can be 50-80% of Ctotal!
    (wires are closer together than to ground planes)
```

### 1.2 Why Crosstalk Gets Worse at Advanced Nodes

```ascii-graph
Wire pitch scaling:
  130nm: M1 pitch = 340nm, spacing = 200nm
  65nm:  M1 pitch = 190nm, spacing = 100nm
  7nm:   M1 pitch = 28nm,  spacing = 14nm

Coupling capacitance ∝ (H × L) / S
  H: wire height (stays roughly constant or grows — taller for lower R)
  L: wire length (coupling run length)
  S: wire spacing (shrinks aggressively)

  Result: Cc/Ctotal ratio increases dramatically
  At 7nm, coupling dominates total capacitance on lower metals

Also: higher resistance wires → RC time constant grows → noise lingers longer
```

### 1.3 Crosstalk Types

1. Functional crosstalk (glitch noise):
   - Aggressor switches, victim is quiet
   - Coupling injects noise voltage on victim
   - If noise exceeds noise margin → functional failure

2. Timing crosstalk (delay impact):
   - Aggressor and victim switch simultaneously
   - Same direction (SI): effective Cc reduces → speed-up
   - Opposite direction (OI): effective Cc increases → slow-down
   - Can cause timing violations (setup or hold)

3. Dynamic noise (IR drop induced):
   - Simultaneous switching output (SSO) noise
   - Many signals switching → current spike → VDD droops/GND bounces
   - Not strictly "crosstalk" but a related SI (signal integrity) effect

---

## 2. Crosstalk Noise Analysis

### 2.1 Glitch Noise Calculation

```ascii-graph
Simplified model (RC network):

  Aggressor: Step transition from 0 → VDD
  Victim: Quiet (driven to 0 or floating)

  Noise voltage on victim:
    V_noise_peak ≈ VDD × Cc / (Cc + Cv + Cd)

  Where:
    Cc: coupling capacitance
    Cv: victim ground capacitance
    Cd: victim driver output capacitance (if driven)

  For a strongly driven victim (low impedance driver):
    V_noise_peak ≈ VDD × Cc / (Cc + Cv) × (Ragg / (Ragg + Rv_driver))
    Much smaller — strong driver "holds" the victim

  For a floating victim (e.g., dynamic node or bus in tri-state):
    V_noise_peak ≈ VDD × Cc / (Cc + Cv)
    Can be very large! (no driver to fight the noise)
```

### 2.2 Noise Propagation

After noise is induced, it propagates through downstream logic:

Noise → gate input → if noise > switching threshold →
gate output switches → propagates further

**Noise immunity levels:**
   - Level 1 (safe):       Noise < 0.1 × VDD → no functional impact
   - Level 2 (marginal):   0.1 × VDD < Noise < NM (noise margin) → may cause issues at worst corner
   - Level 3 (critical):   Noise > NM → can flip downstream gate → functional failure

**Noise budget allocation:**
   - Total noise margin:           NM ≈ 0.35V (for VDD = 0.7V)
   - Allocated to crosstalk:       ~40% = 0.14V
   - Allocated to power supply:    ~30% = 0.105V
   - Allocated to process variation: ~30% = 0.105V

### 2.3 Crosstalk Noise in STA Tools

```verilog
PrimeTime SI / Tempus SI flow:
  1. Extract parasitics with coupling capacitances (SPEF with coupled_net sections)
  2. Tool identifies aggressor-victim pairs
  3. Calculates noise bump at victim pin
  4. Propagates noise through combinational logic
  5. Reports noise violations (exceeding threshold)

SPEF coupling entry:
  *D_NET victim_net 0.15    // total cap = 0.15pF
  *CONN
  ...
  *CAP
  1 victim_net:1 0.03       // ground cap
  2 victim_net:2 0.02       // ground cap
  *R
  ...
  *COUPLED_NET aggressor_net
  victim_net:1 aggressor_net:5 0.008  // coupling cap = 8fF
```

---

## 3. Crosstalk Delay Impact

### 3.1 Miller Effect on Delay

```ascii-graph
When aggressor and victim switch simultaneously:

  Same direction (aggressor and victim both rise):
    Effective Cc = 0 (no ΔV across coupling cap → no charge transfer)
    Victim sees LESS capacitance → FASTER

  Opposite direction (aggressor rises, victim falls):
    Effective Cc = 2×Cc (ΔV across coupling cap = 2×VDD)
    Victim sees MORE capacitance → SLOWER

  General Miller factor:
    Cc_effective = Cc × (1 - ΔVaggressor/ΔVvictim)

    Same direction: ΔVagg/ΔVvic = 1  → Cc_eff = 0
    Victim static:  ΔVagg/ΔVvic = ∞  → Cc_eff = Cc (noise)
    Opposite dir:   ΔVagg/ΔVvic = -1 → Cc_eff = 2×Cc

Delay impact example:
  Base delay = R × (Cground + Cc) = 100ps
  With opposite switching: delay = R × (Cground + 2×Cc)
  If Cc = Cground: delay = R × 3×Cground = 150ps  (50% increase!)
```

### 3.2 SI-Aware STA

```ascii-graph
PrimeTime SI methodology:

  Setup analysis (max delay):
    - Data path: apply worst-case crosstalk (opposite switching → slow down)
    - Clock path: apply best-case crosstalk (same switching → speed up)
    - This maximizes the delay difference → pessimistic for setup

  Hold analysis (min delay):
    - Data path: apply best-case crosstalk (same switching → speed up)
    - Clock path: apply worst-case crosstalk (opposite switching → slow down)

  Filtering:
    - Not all aggressors switch simultaneously in practice
    - Tool uses timing windows to filter infeasible scenarios
    - If aggressor's switching window doesn't overlap victim's → no crosstalk
```

### 3.3 Crosstalk Fixing Techniques

```ascii-graph
Technique              | How it helps              | Cost
-----------------------|---------------------------|------------------
Increase spacing       | Cc ∝ 1/S → less coupling  | More routing area
Wire shielding         | GND wire between signals  | Doubles wire usage
Layer change           | Move to different layer    | May affect timing
Net ordering           | Route non-switching nets   | Routing complexity
                       |   between aggressors       |
Buffer insertion       | Lower driver impedance     | Area, power
Downsizing aggressor   | Slower aggressor → less    | May hurt aggressor
                       |   noise injection          |   timing
NDR (Non-Default Rules)| Wider spacing/width for    | Routing resource
                       |   critical nets            |
```

---

## 4. Electromigration (EM)

### 4.1 What is Electromigration

```ascii-graph
Electron current exerts force on metal atoms (momentum transfer).
At high current density, atoms physically move in the direction of
electron flow (opposite to conventional current flow).

  Atom migration → voids form (open circuit risk)
                 → hillocks form (short circuit risk)

  ←── Electron flow ────
  ────────────→ Atom migration
  
  ┌────────────────────────────────┐
  │    Void        Cu wire      Hillock │
  │   ┌──┐                     ┌──┐  │
  │───┘  └─────────────────────┘  └──│
  │                                    │
  └────────────────────────────────┘
       ↑ Open circuit risk    ↑ Short circuit risk
```

### 4.2 Black's Equation

```ascii-graph
MTTF = A × J^(-n) × exp(Ea / (k × T))

  MTTF: Mean Time To Failure
  A:    Material/geometry constant
  J:    Current density (A/cm²)
  n:    Current density exponent (typically 1-2)
  Ea:   Activation energy (eV)
        Cu (grain boundary): Ea ≈ 0.7-0.9 eV
        Cu (interface):      Ea ≈ 0.8-1.0 eV
        Al:                  Ea ≈ 0.5-0.7 eV
  k:    Boltzmann constant (8.617 × 10⁻⁵ eV/K)
  T:    Temperature (K)

Numerical example:
  Cu wire at 105°C (378K), J = 2 MA/cm²:
  MTTF = A × (2e6)^(-2) × exp(0.85 / (8.617e-5 × 378))
       = A × 2.5e-13 × exp(26.1)
       = A × 2.5e-13 × 2.2e11
       = A × 0.055

  At J = 1 MA/cm² (half the current density):
  MTTF = A × 1e-12 × exp(26.1) = A × 0.22  → 4× longer lifetime!

  Doubling current density → 4× shorter lifetime (n=2)
```

### 4.3 EM Design Rules

```ascii-graph
Current density limits (typical, per metal layer):

  Layer  | Width   | DC limit    | RMS limit   | Peak limit
  -------|---------|-------------|-------------|------------
  M1     | 28nm    | 0.5 mA/μm  | 1.0 mA/μm  | 5.0 mA/μm
  M4     | 56nm    | 1.0 mA/μm  | 2.0 mA/μm  | 10 mA/μm
  M10    | 280nm   | 3.0 mA/μm  | 6.0 mA/μm  | 30 mA/μm
  RDL    | 1.6μm   | 10 mA/μm   | 20 mA/μm   | 100 mA/μm

  DC limit:  Average current (for unidirectional current)
  RMS limit: For bidirectional current (clock, data)
  Peak limit: Instantaneous maximum

  Via EM limits (per via cut):
    Single-cut via: ~0.1-0.2 mA per cut
    This is why multi-cut vias are critical for reliability!

Width calculations:
  Required wire width = I_avg / J_limit
  Example: 2mA signal on M1 → W = 2mA / 0.5mA/μm = 4μm minimum width
  (This is why power straps must be wide!)
```

**Worked wire-level EM check (PnR — place-and-route — view):**

**Black's Equation:**
```text
  MTTF = A * (J)^(-n) * exp(Ea / (k*T))
  
  Where:
    MTTF = Mean Time To Failure
    A = constant (process dependent)
    J = current density (A/cm^2)
    n = current density exponent (typically 1-2)
    Ea = activation energy (~0.7 eV for Al, ~0.9 eV for Cu grain boundary)
    k = Boltzmann constant (8.617e-5 eV/K)
    T = temperature (Kelvin)
```

**Numerical Example:**
```ascii-graph
Given:
  Wire on M3 at 7nm: width = 36nm, thickness = 50nm
  Wire area = 36nm * 50nm = 1800 nm^2 = 1.8e-11 cm^2
  
  EM limit for M3 = 1 MA/cm^2 = 1e6 A/cm^2
  Max DC current = J_limit * Area = 1e6 * 1.8e-11 = 18 uA (per wire!)
  
  For AC (RMS) current: limit is typically 3-10x higher due to
  self-healing during current reversal.
  
  For power straps (much wider, e.g., 2 um):
  Area = 2000nm * 100nm = 2e-9 cm^2
  Max current = 1e6 * 2e-9 = 2 mA per strap
  
  With 100 parallel straps: 200 mA total → need to verify this is sufficient.
```

**EM by metal layer:**

| Metal Layer | Typical EM Limit (DC) | Wire Width | Max Current/Wire |
|-------------|----------------------|------------|------------------|
| M1          | 0.5 MA/cm^2          | 36nm       | ~9 uA            |
| M4          | 1.0 MA/cm^2          | 40nm       | ~20 uA           |
| M8          | 2.0 MA/cm^2          | 200nm      | ~400 uA          |
| M12 (thick) | 3.0 MA/cm^2          | 2000nm     | ~30 mA           |



**Electromigration Limits:**

```verilog
  For each power grid segment and via:
    J = I / A  (current density)

  where A = cross-sectional area of the wire or via

  EM limits (typical, 7nm Cu):
    DC:  J_max = 1-3 MA/cm^2 (depends on layer, temperature, lifetime target)
    AC (RMS): J_max_rms = 5-15 MA/cm^2 (self-healing extends limit)

  Checking methodology:
  1. Compute average current per power grid segment (from activity-aware
     power analysis -- requires VCD or SAIF switching activity data)
  2. For each wire segment: J = I_avg / (width * thickness)
  3. For each via: J = I_avg / (N_cuts * via_area)
  4. Compare J against J_max for that layer and temperature
  5. Flag any segment where J > J_max

  Example (M4 power strap, 7nm):
    Width = 0.2 um, thickness = 0.08 um
    Area = 0.2 * 0.08 = 0.016 um^2 = 1.6e-9 cm^2
    I_segment = 2 mA (from power analysis)
    J = 2e-3 / 1.6e-9 = 1.25 MA/cm^2
    J_max (M4 DC, 105C, 10yr) = 1.0 MA/cm^2
    -> VIOLATION: need wider strap or additional parallel straps
    
  Fix: increase strap width to 0.3 um:
    J = 2e-3 / (0.3 * 0.08 * 1e-8) = 0.83 MA/cm^2 -> PASS

  Via EM check:
    V4 via: area per cut = 0.04 * 0.04 = 1600 nm^2 = 1.6e-10 cm^2
    I_via = 0.5 mA, N_cuts = 2 (double-cut via)
    J_via = 0.5e-3 / (2 * 1.6e-10) = 1.56 MA/cm^2
    J_max_via = 2.0 MA/cm^2 -> PASS
```



### 4.4 Self-Heating EM

```ascii-graph
At advanced nodes, wire self-heating exacerbates EM:

  Wire temperature = T_ambient + ΔT_self_heat
  
  ΔT_self_heat = I²R × Rth
  
  Rth: thermal resistance to heat sink
  Low-k dielectrics have POOR thermal conductivity (0.15 W/m·K vs SiO2 = 1.4)
  → Heat generated in wire cannot escape easily
  → Local temperature rises significantly
  → EM lifetime further reduced (exponential T dependence)

  At 7nm: ΔT_self_heat can be 10-30°C for heavily loaded metal lines
  This must be accounted for in EM signoff analysis
```

---

## 5. IR Drop Analysis

*Scope: the PnR/implementation view — grid models, analysis, fixing. Signoff criteria and power-integrity tool flow: [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md) §3.*

### 5.1 Static IR Drop

```ascii-graph
Static IR drop: Voltage drop due to average DC current through resistive power grid

  V_drop = I_avg × R_grid

  Where R_grid is the effective resistance from pad/bump to the cell

  ┌─────────── VDD pad ─────────────┐
  │                                  │
  │    R1      R2       R3      R4   │
  ├────┤├─────┤├──────┤├──────┤├────┤
  │    I1      I2       I3      I4   │
  │    ↓       ↓        ↓       ↓    │  Standard cells drawing current
  ├────┤├─────┤├──────┤├──────┤├────┤
  │                                  │
  └─────────── VSS pad ─────────────┘

  Voltage at each cell: V_cell = VDD - Σ(I×R along path from pad)

  Typical budget:
    Static IR drop ≤ 3-5% of VDD
    At VDD = 0.7V → max drop ≈ 21-35 mV
    This translates to ~5-10% performance degradation
```

### 5.2 Dynamic IR Drop

```ascii-graph
Dynamic IR drop: Voltage drop due to instantaneous switching current

  Much worse than static because:
  1. Peak current >> average current (during clock edges)
  2. Package inductance (Ldi/dt noise)
  3. Decoupling capacitance may be insufficient
  4. Localized hotspots where many cells switch simultaneously

  V_drop_dynamic = L × (dI/dt) + I_peak × R_grid

  Clock edge: All flip-flops switch → enormous instantaneous current
  Worst case: First few clock cycles after power-up or wake-from-sleep

  Dynamic IR drop budget:
    ≤ 8-10% of VDD for nominal operation
    May exceed 15% during worst-case events → clock frequency derate needed
```

### 5.3 IR Drop Analysis Flow

```text
Tools: RedHawk (Synopsys), Voltus (Cadence), MVSIM (Apache)

Static IR drop flow:
  1. Read DEF (physical design database)
  2. Read LEF/Tech LEF (layer info, via resistance)
  3. Read power grid (PG netlist extraction)
  4. Apply current profile:
     - From SAIF (switching activity) + Liberty (cell current)
     - Or from VCD-based power analysis
  5. Solve resistive network: I = G × V (conductance matrix)
  6. Report voltage map: IR drop at each instance

Dynamic IR drop flow:
  1. Read all static inputs + parasitic extraction
  2. Read VCD or FSDB (switching activity per cycle)
  3. Time-domain simulation of power grid (RLC network)
  4. Report worst-case voltage drop per cell, per cycle
  5. Identify hotspot regions and timing

Signoff criteria:
  Static:  Max IR drop < 30 mV (typical target)
  Dynamic: Max IR drop < 50-70 mV (at worst-case event)
  EM:      All nets pass current density limits for target lifetime (10 years)
```

### 5.4 IR Drop Fixing

```ascii-graph
1. Widen power straps (reduce R):
   Wider metal → lower resistance → less voltage drop
   But consumes routing resources

2. Add more power vias:
   Vias are resistive bottlenecks → more vias = lower R
   Multi-cut vias: also improve EM reliability

3. Add decoupling capacitors (decap cells):
   Charge reservoir near switching cells
   Supplies instantaneous current → reduces Ldi/dt noise
   Typical: fill empty spaces with decap filler cells
   
   Effective radius: decap helps within ~50-100 μm
   Beyond that, wire resistance to decap is too high

4. Optimize placement:
   Spread high-switching cells to avoid hotspots
   Place high-power blocks near power pads/bumps

5. Add power pads/bumps:
   More supply connections → current distributed → lower R
   Flip-chip bumps: much better than wire-bond (lower L, more connections)
```

---

### 5.5 Grid-Level IR Models — Worked Examples (from the PnR view)

**Power Grid Modeling (Resistance Network):**

```ascii-graph
  The power delivery network is modeled as a resistive mesh:

  VDD pad ──[R_bump]──┬──[R_M10]──┬──[R_M10]──┬── ... (top metal grid)
                       |           |           |
                    [R_via]     [R_via]     [R_via]
                       |           |           |
                    [R_M8]────[R_M8]────[R_M8]── ... (intermediate grid)
                       |           |           |
                    [R_via]     [R_via]     [R_via]
                       |           |           |
                    [R_M5]────[R_M5]────[R_M5]── ... (lower grid)
                       |           |           |
                    cell_i      cell_j      cell_k  (current sinks)

  Each R value is computed from:
    R = rho * L / (W * T)  = R_sheet * (L / W)
    
    where R_sheet = sheet resistance (ohm/square)
    
  Typical R_sheet values (7nm):
    M1:   ~100 mohm/sq   (thin, narrow)
    M4:   ~50 mohm/sq
    M6:   ~20 mohm/sq
    M8:   ~5 mohm/sq
    M10:  ~1 mohm/sq     (thick, wide)

  Via resistance per cut:
    V1 (M1-M2): ~5 ohm
    V4 (M4-M5): ~2 ohm
    V8 (M8-M9): ~0.5 ohm
    Multi-cut: R_eff = R_via / N_cuts

  The grid has millions of nodes (one per intersection).
  Static IR drop solves: V = V_source - R_matrix * I_vector
  where R_matrix is the full resistance network and I_vector is the
  current drawn at each node. Solved via sparse matrix techniques
  (conjugate gradient, multigrid) in tools like RedHawk/Voltus.
```

**Static IR Drop:**
```verilog
  Average current consumption -> DC voltage drop across power grid.

  V_drop = I_avg * R_grid

  Measured at every node of the power grid.
  Output: voltage contour map showing drop from supply to each cell.

  Target: V_drop < 3-5% of VDD
    At VDD = 0.75V: max drop = 22.5-37.5 mV

  Current calculation per cell:
    I_avg = P_avg / VDD = (C_total * VDD^2 * f * alpha) / VDD
          = C_total * VDD * f * alpha

    where:
      C_total = total switched capacitance (cell + wire)
      f = clock frequency
      alpha = activity factor (toggle rate)

  Example for a clock buffer driving 50 FFs:
    C_load = 50 * 2 fF = 100 fF (cell input caps)
    C_wire = 200 fF (clock tree wire)
    C_total = 300 fF
    VDD = 0.75V, f = 1 GHz, alpha = 1.0 (clock toggles every cycle)
    I_avg = 300e-15 * 0.75 * 1e9 * 1.0 = 0.225 mA
    
    If this buffer is 5 squares from the power pad on M8 (R_sheet = 5 mohm/sq):
    R_path = 5 * 0.005 = 0.025 ohm
    V_drop_static = 0.225e-3 * 0.025 = 5.6 uV (negligible for one buffer)
    
    But 10,000 such buffers: I_total = 2.25 A
    R_grid_effective ≈ 0.05 ohm (parallel paths)
    V_drop = 2.25 * 0.05 = 112.5 mV (15% of VDD -- WAY over budget!)
    -> Need more power straps or additional grid layers.
```

**Dynamic IR Drop:**
```text
  Considers simultaneous switching of many cells (worst-case transient).

  V_drop_dynamic = L * di/dt + R * i(t)

  The di/dt term (inductance) causes supply bounce (Ldi/dt noise).
  Much worse than static IR drop -- can be 2-5x larger.

  Analysis requires:
    1. VCD (Value Change Dump) from simulation for switching activity
    2. Power grid model (R, L, C network)
    3. Transient simulation of power grid
    4. Package model (bond wire / bump inductance, typically 0.1-1 nH)

  Dynamic IR drop scenario (worst case):
    - All clock tree buffers switch simultaneously at the clock edge
    - Clock tree power: 30% of total dynamic power
    - At 1 GHz: di/dt peak ≈ 10-50 A/ns
    - With L_package = 0.5 nH: V_bounce = 0.5e-9 * 30e9 = 15V !! 
      (This is clearly unrealistic -- the on-die decap absorbs most of this)
    
    With sufficient on-die decoupling capacitance (C_decap):
    - C_decap provides local charge: delta_V = I * dt / C_decap
    - Need: C_decap > I_peak * t_switch / delta_V_budget
    - For I_peak = 5A, t_switch = 100ps, delta_V = 50mV:
      C_decap > 5 * 100e-12 / 0.05 = 10 nF

  Output: time-varying voltage map, identify hotspots and worst-case time windows.
```

**IR Drop Map (ASCII representation):**
```ascii-graph
  +------+------+------+------+
  | 15mV | 20mV | 25mV | 18mV |   Voltage drop values at grid nodes
  +------+------+------+------+
  | 22mV | 30mV | 38mV!| 28mV |   38mV exceeds 37.5mV budget!
  +------+------+------+------+   -> Need more power straps or
  | 18mV | 25mV | 32mV | 22mV |     decap cells in that region
  +------+------+------+------+
  | 10mV | 15mV | 20mV | 12mV |
  +------+------+------+------+
```

---



## 6. Power Grid Design

### 6.1 Power Grid Structure

```ascii-graph
Hierarchical power grid:

  Top metals (M10-M12): Wide straps, carry bulk current
    - Width: 2-10 μm
    - Pitch: 20-50 μm
    - Carry current from bumps to lower metals

  Middle metals (M5-M9): Intermediate distribution
    - Width: 0.5-2 μm
    - Pitch: 5-20 μm

  Lower metals (M1-M4): Connect to standard cells
    - M1: horizontal power rails within cells
    - Cell internal VDD/VSS rails: ~100nm width at 7nm
    - Connected to upper grid by via stacks

  Cross-section:
    M12: ═══════VDD═══════  ═══VSS═══════  (wide straps)
          ┃     ┃     ┃     ┃     ┃     ┃   (via stacks)
    M8:  ═╩═VDD═╩═VSS═╩═VDD═╩═VSS═╩═VDD═╩═  (mesh)
          ┃  ┃  ┃  ┃  ┃  ┃  ┃  ┃  ┃  ┃  ┃
    M1:  ─VDD─VSS─VDD─VSS─VDD─  (cell rails)
```

### 6.2 Grid Design Parameters

```ascii-graph
Key design decisions:
  1. Strap width and pitch (determines R and routing blockage)
  2. Number of metal layers used for power (vs signal routing)
  3. Via stack configuration (via arrays at intersections)
  4. Mesh vs stripe topology:
     
     Mesh:   Grid in both X and Y → redundant paths
             More robust to local defects
             More routing blockage
     
     Stripe: Parallel straps in one direction per layer
             Alternating layers → orthogonal coverage
             Less routing blockage, easier to route around

Grid resistance estimation:
  R_sheet = ρ / t  (sheet resistance)
  For Cu at 7nm, M10 (t ≈ 500nm): R_sheet ≈ 35 mΩ/□
  
  A 2μm wide strap, 100μm long:
  R = R_sheet × (L/W) = 35 mΩ × (100/2) = 1.75 Ω
  
  With 1mA current: V_drop = 1.75 mV (acceptable)
  With 100mA current: V_drop = 175 mV (way too high!)
  → Need many parallel straps for high-current regions
```

### 6.3 Decoupling Capacitance

```ascii-graph
Purpose: Local charge storage to supply transient current

Sources of decap:
  1. Explicit decap cells (filler cells with MOS cap):
     - Large gate area → high C per unit area
     - Typical: 2-5 fF/μm² at 7nm
     
  2. Intrinsic decap (existing logic):
     - Gate capacitance of inactive transistors
     - Typically 50-70% of total chip decap comes from this
     
  3. Package decap:
     - Discrete capacitors on package substrate
     - Effective for low-frequency noise (< 100 MHz)
     - Too far from chip for high-frequency noise (> 500 MHz)

Sizing requirement:
  Q = C × ΔV  →  $C = I_{peak} \times \Delta t / \Delta V_{allowed}$

  Example:
    I_peak = 10A (clock edge switching)
    Δt = 100ps (current pulse duration)
    ΔV_allowed = 35mV (5% of 0.7V)
    
    C = 10 × 100e-12 / 35e-3 = 28.6 nF
    
    At 5 fF/μm², need 5.72 × 10⁶ μm² = 5.72 mm²
    For a 100mm² die → 5.7% area for decap (reasonable)
```

---

**Typical Power Grid Design Parameters:**

```ascii-graph
  Mesh density (for a 10mm x 10mm die, 7nm, 2W total power):
  
  M10 (top metal):  stripes every 20 um, width 5 um
    -> 500 stripes per direction, each 10mm long
    -> R per stripe = R_sheet * L/W = 0.001 * (10000/5) = 2 ohm
    -> 500 parallel: R_eff = 2/500 = 4 milliohm
    -> IR drop from M10 alone: 2.67A * 0.004 = 10.7 mV (manageable)

  M8: stripes every 10 um, width 2 um
    -> 1000 stripes per direction
    -> R per stripe = 0.005 * (10000/2) = 25 ohm
    -> 1000 parallel: R_eff = 25 milliohm

  M6: stripes every 5 um, width 1 um
    -> Finer grid for local distribution

  Via arrays at intersections: 4x4 array of V8-V9 vias
    -> R_via_array = R_via / 16 ≈ 0.03 ohm (negligible)

  Decap cells: fill 10% of whitespace
    -> Typical: 0.5-1 nF per mm^2 of decap density
    -> For 100 mm^2 die: 50-100 nF total on-die decap

  Total IR drop budget allocation:
    Package + bumps:  10 mV
    M10-M9 mesh:      10 mV
    M8-M6 stripes:    10 mV
    M5-M1 rails:       5 mV
    Dynamic (Ldi/dt):  5 mV (after decap)
    ─────────────────────
    Total:            40 mV (5.3% of 0.75V, within budget)
```

---



## 7. Thermal Analysis

*Scope: implementation-side thermal modeling and mitigation. Budgeting, thermal runaway math, and DTM policy: [Power_Analysis_and_Signoff](../02_Power_and_Low_Power/05_Power_Analysis_and_Signoff.md) §6.*

### 7.1 Heat Generation and Dissipation

```ascii-graph
Heat generation:
  P_total = P_dynamic + P_leakage
  Heat flux: q = P / A  (W/cm²)

  Modern SoCs: 50-100 W/cm² average, 300-500 W/cm² hotspots
  Compare: hot plate = ~10 W/cm², rocket nozzle = ~500 W/cm²

Heat dissipation path:
  Junction → Die → TIM → Heat spreader → Heat sink → Air

  Thermal resistance model (series):
  T_junction = T_ambient + P × (Θ_jc + Θ_cs + Θ_sa)

  Θ_jc: Junction-to-case (~0.1-0.5 °C/W)
  Θ_cs: Case-to-sink (~0.1-0.5 °C/W)
  Θ_sa: Sink-to-ambient (~0.5-5 °C/W)

  Example:
    P = 100W, Θ_ja = 0.3 °C/W total, T_ambient = 25°C
    T_junction = 25 + 100 × 0.3 = 55°C  (well within 125°C limit)

    But with Θ_ja = 1.0 °C/W:
    T_junction = 25 + 100 × 1.0 = 125°C  (at the thermal limit!)
```

### 7.2 Thermal Hotspots

```ascii-graph
Hotspot: Local region with much higher temperature than average

Causes:
  - High activity blocks (CPU core during benchmark)
  - Power-dense IP (PLL, SerDes, DDR PHY)
  - Stacked vias concentrating heat
  - Poor local heat dissipation (under macro, near die edge)

Impact:
  - Leakage increases exponentially with temperature
    I_leak ∝ exp(-Ea/kT) → hotspot creates positive feedback loop!
  - EM lifetime decreases exponentially with temperature
  - Timing varies (temperature inversion at low VDD)
  - Reliability (BTI, HCI) worsens at high temperature

Thermal runaway:
  If power increases faster than heat can dissipate:
  Higher T → more leakage → more power → higher T → ...
  This positive feedback can destroy the chip
  Protection: thermal diode + throttling circuit (DVFS, clock gating)
```

### 7.3 On-Chip Thermal Sensors

```ascii-graph
Thermal diode:
  - Forward-biased PN junction
  - VBE decreases ~2 mV/°C (negative temperature coefficient)
  - ADC reads VBE → digital temperature value

  Accuracy: ±1-3°C after calibration
  Placement: Near each major hotspot (CPU core, GPU, memory controller)

Thermal management:
  1. T < T_normal: Full performance
  2. T_normal < T < T_throttle: Begin DVFS (reduce voltage/frequency)
  3. T_throttle < T < T_critical: Aggressive throttling
  4. T > T_critical: Emergency shutdown (protect silicon)

  Typical limits:
    T_throttle ≈ 90-100°C
    T_critical ≈ 105-125°C
```

---

## 8. Reliability Signoff Methodology

### 8.1 Reliability Mechanisms

```text
Mechanism         | Root Cause                    | Impact
------------------|-------------------------------|------------------
NBTI              | Hole trapping in PMOS oxide   | Vtp increases over time
PBTI              | Electron trapping in NMOS     | Vtn increases over time
                  |   (worse with high-k)         |
HCI               | Hot carrier injection         | Vth shift, gm degradation
TDDB              | Time-dependent dielectric     | Gate oxide breakdown
                  |   breakdown                   |
EM                | Atom migration in metal       | Wire open/short circuit
SM (Stress Mig.)  | Mechanical stress in metal    | Void formation
ESD               | Electrostatic discharge       | Oxide rupture, junction damage
```

### 8.2 Aging Analysis (BTI + HCI)

**BTI (Bias Temperature Instability):**
   - NBTI (Negative BTI): Affects PMOS, VGS < 0 (normal operation)
   - ΔVth ∝ t^n × exp(-Ea/kT)    (n ≈ 0.16-0.25, power law)

10-year aging at 125°C: ΔVth ≈ 30-50 mV (PMOS) → Performance degradation of ~5-10% → Must be included in timing signoff

PBTI (Positive BTI): Affects NMOS with high-k dielectric
Similar mechanism but with electrons instead of holes
10-year aging: ΔVth ≈ 10-30 mV (NMOS)

**Aging-aware STA (static timing analysis):**
   - Library provides aged cell models (fresh + 10-year)
   - PrimeTime: read_lib fresh.lib; read_lib -age aged.lib
   - Signoff against aged timing ensures chip meets spec over lifetime

### 8.3 Complete Reliability Signoff Checklist

```text
□ EM analysis (all metal layers + vias):
  - DC EM for unidirectional nets (power, ground)
  - AC EM (RMS) for bidirectional nets (signal, clock)
  - Peak current check
  - Via EM with multi-cut via requirements

□ IR drop analysis:
  - Static IR drop < 3-5% VDD
  - Dynamic IR drop < 8-10% VDD (with VCD-based analysis)
  - Power grid EM check

□ Thermal analysis:
  - Junction temperature map
  - Hotspot identification
  - Thermal throttling specification

□ ESD compliance:
  - All I/O pads protected (HBM ≥ 2kV, CDM ≥ 500V)
  - Power clamps between all supply domains
  - CDM protection for cross-domain signals

□ Aging analysis:
  - NBTI/PBTI margin included in timing analysis
  - HCI check on always-on paths
  - TDDB voltage limits respected

□ DRC/LVS/ERC clean:
  - All physical verification rules passed
  - Antenna rule compliance
  - Density checks (metal, poly, OD)
```

---

