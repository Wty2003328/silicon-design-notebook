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
9. Interview Q&A (20+ Questions)

---

## 1. Crosstalk Fundamentals

### 1.1 Coupling Capacitance Model

```
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

```
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

```
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
   - Not strictly "crosstalk" but a related SI effect
```

---

## 2. Crosstalk Noise Analysis

### 2.1 Glitch Noise Calculation

```
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

```
After noise is induced, it propagates through downstream logic:

  Noise → gate input → if noise > switching threshold →
          gate output switches → propagates further

Noise immunity levels:
  Level 1 (safe):       Noise < 0.1 × VDD → no functional impact
  Level 2 (marginal):   0.1 × VDD < Noise < NM → may cause issues at worst corner
  Level 3 (critical):   Noise > NM → can flip downstream gate → functional failure

Noise budget allocation:
  Total noise margin:           NM ≈ 0.35V (for VDD = 0.7V)
  Allocated to crosstalk:       ~40% = 0.14V
  Allocated to power supply:    ~30% = 0.105V
  Allocated to process variation: ~30% = 0.105V
```

### 2.3 Crosstalk Noise in STA Tools

```
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

```
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

```
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

```
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

```
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

```
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

```
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

### 4.4 Self-Heating EM

```
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

### 5.1 Static IR Drop

```
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

```
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

```
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

```
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

## 6. Power Grid Design

### 6.1 Power Grid Structure

```
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

```
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

```
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
  Q = C × ΔV  →  C = I_peak × Δt / ΔV_allowed

  Example:
    I_peak = 10A (clock edge switching)
    Δt = 100ps (current pulse duration)
    ΔV_allowed = 35mV (5% of 0.7V)
    
    C = 10 × 100e-12 / 35e-3 = 28.6 nF
    
    At 5 fF/μm², need 5.72 × 10⁶ μm² = 5.72 mm²
    For a 100mm² die → 5.7% area for decap (reasonable)
```

---

## 7. Thermal Analysis

### 7.1 Heat Generation and Dissipation

```
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

```
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

```
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

```
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

```
BTI (Bias Temperature Instability):
  NBTI (Negative BTI): Affects PMOS, VGS < 0 (normal operation)
    ΔVth ∝ t^n × exp(-Ea/kT)    (n ≈ 0.16-0.25, power law)
    
    10-year aging at 125°C: ΔVth ≈ 30-50 mV (PMOS)
    → Performance degradation of ~5-10%
    → Must be included in timing signoff

  PBTI (Positive BTI): Affects NMOS with high-k dielectric
    Similar mechanism but with electrons instead of holes
    10-year aging: ΔVth ≈ 10-30 mV (NMOS)

Aging-aware STA:
  Library provides aged cell models (fresh + 10-year)
  PrimeTime: read_lib fresh.lib; read_lib -age aged.lib
  Signoff against aged timing ensures chip meets spec over lifetime
```

### 8.3 Complete Reliability Signoff Checklist

```
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

## 9. Interview Q&A

**Q1: What is crosstalk and what causes it?**

Crosstalk is unintended signal coupling between adjacent wires through parasitic coupling
capacitance. When an aggressor wire switches, current flows through the coupling
capacitance and induces a voltage change on the neighboring victim wire. At advanced nodes
(7nm and below), coupling capacitance can be 50-80% of total wire capacitance because wire
spacing has shrunk much faster than wire height.

**Q2: Explain the Miller effect in crosstalk delay.**

When aggressor and victim switch in opposite directions, the voltage change across the
coupling capacitance is 2×VDD, making the effective coupling capacitance 2×Cc (Miller
factor = 2). This additional load slows the victim. When they switch in the same direction,
the effective coupling is 0 (Miller factor = 0), speeding up the victim. The worst case
for setup timing is opposite switching; for hold timing, it's same-direction switching.

**Q3: How does SI-aware STA work?**

PrimeTime SI reads the design with coupling capacitances from parasitic extraction. For
each data path, it identifies aggressor nets whose switching windows overlap with the
victim. For setup analysis, it assumes worst-case (opposite switching) on data paths and
best-case (same switching) on clock paths. It uses timing windows to filter out physically
impossible aggressor-victim combinations, avoiding excessive pessimism.

**Q4: Write Black's equation and explain each term.**

MTTF = A × J^(-n) × exp(Ea/(kT)). A is a material constant, J is current density (higher
J → shorter life), n is the current density exponent (1 for void nucleation, 2 for void
growth), Ea is activation energy (higher → more EM-resistant, Cu ~0.7-0.9 eV vs Al
~0.5-0.7 eV), k is Boltzmann constant, T is temperature (exponential dependence — 10°C
increase can halve lifetime).

**Q5: What is the difference between static and dynamic IR drop?**

Static IR drop is the average voltage drop due to DC current flowing through the resistive
power grid — it's constant over time. Dynamic IR drop is the instantaneous voltage drop
during switching events (especially clock edges), which can be much worse due to high peak
currents and package inductance (Ldi/dt). Dynamic IR drop is typically 2-3× worse than
static and causes localized timing failures.

**Q6: How do you fix IR drop violations?**

(1) Widen power straps to reduce grid resistance. (2) Add more power vias (especially
multi-cut vias). (3) Insert decoupling capacitor cells near hotspot regions. (4) Add more
power pads/bumps for shorter current paths. (5) Redistribute high-power cells to avoid
current concentration. (6) Increase metal layer allocation for power grid.

**Q7: What is self-heating EM and why is it a concern at advanced nodes?**

At advanced nodes, metal wires are so thin and surrounded by low-k dielectric (poor
thermal conductor) that Joule heating significantly raises the wire temperature. This
elevated temperature is fed back into Black's equation, dramatically reducing EM lifetime.
A wire might pass EM at the nominal temperature but fail when self-heating adds 10-30°C.
Modern EM signoff tools (RedHawk, Voltus) include self-heating models.

**Q8: How do you design the power grid for an SoC?**

Start from power budget (total current per domain). Size top-metal straps to carry the
required current within EM limits. Create a mesh with redundant paths. Use via arrays at
strap intersections. Size lower-metal connections to standard cells. Run static IR drop
analysis and iterate strap width/pitch. Add decap cells to reduce dynamic IR drop. Verify
EM on all grid elements. Typical guideline: power grid uses 10-20% of routing resources.

**Q9: What is NBTI and how does it affect timing signoff?**

Negative Bias Temperature Instability affects PMOS transistors during normal operation
(VGS < 0). Holes trapped in the gate oxide increase Vth over time, degrading drive
current and slowing the transistor. After 10 years at 125°C, ΔVth can be 30-50 mV.
Timing signoff must use "aged" library models that include this degradation, ensuring the
chip still meets timing at end-of-life.

**Q10: How do you prevent EM in clock networks?**

Clock nets carry bidirectional current (switching 0→1→0 every cycle) at high frequency.
Use AC/RMS EM limits (higher than DC limits). Ensure clock buffers/inverters have adequate
drive strength (reduces current per wire). Use NDR (double-width, double-spacing) for
clock wires — reduces current density and resistance. Multi-cut vias at every clock buffer
connection. Monitor via EM especially carefully (single-cut vias are weakest links).

**Q11: What is thermal runaway?**

When a chip's leakage power creates a positive feedback loop: higher temperature →
exponentially more leakage → more heat → higher temperature. If heat generation exceeds
dissipation capacity, the temperature rises uncontrollably until the chip fails. Prevention
requires thermal sensors, DVFS throttling, and proper heat sink design. It's especially
dangerous during stress testing or in systems with inadequate cooling.

**Q12: How does crosstalk affect hold timing?**

For hold analysis, the concern is data arriving too fast at the capture register. Crosstalk
that speeds up the data path (same-direction switching, Miller factor = 0) makes data
arrive earlier, potentially causing hold violations. Simultaneously, crosstalk that slows
the clock path (opposite-direction switching on clock wire) makes the capture clock arrive
later. Both effects reduce hold slack. This is why hold violations can appear after routing
even if they weren't present post-synthesis.

**Q13: What is antenna effect and how is it related to reliability?**

During metal etching, long metal lines connected to thin gate oxide accumulate charge from
the plasma process. The charge-to-gate-oxide-area ratio (antenna ratio) must stay within
limits to prevent Fowler-Nordheim tunneling damage to the oxide. If the antenna ratio
exceeds the process limit, the oxide degrades, causing Vth shift and TDDB failure. Fixes:
add protection diodes, break long wires across metal layers, reroute.

**Q14: What is the difference between DC EM and AC EM limits?**

DC EM applies to wires carrying unidirectional current (e.g., power grid where current
always flows from pad to cells). AC EM applies to wires carrying bidirectional current
(e.g., signal and clock nets that switch between 0 and VDD). AC EM limits are typically
2-3× higher than DC limits because the alternating current direction allows partial
healing (atoms migrate back). The relevant metric for AC EM is the RMS current, not
average or peak.

**Q15: How do you calculate the required decoupling capacitance?**

C = I_peak × Δt / ΔV_allowed. Determine peak current from switching activity analysis
(e.g., clock edge current), the duration of the current pulse (typically ~100ps for
clock edge), and the allowed voltage drop (typically 5-10% of VDD). Place decap cells
within ~50-100μm of hotspots (beyond that, wire resistance limits effectiveness). Verify
with dynamic IR drop simulation and iterate.

**Q16: What is via EM and why are multi-cut vias important?**

Single-cut vias have very limited current-carrying capacity (0.1-0.2 mA per cut) due
to their small cross-sectional area. If a signal or power net exceeds this limit, EM
causes void formation at the via interface → open circuit. Multi-cut vias (2, 4, or
more cuts in parallel) divide the current, keeping each via below its EM limit. DFM
rules recommend multi-cut vias wherever space permits. In power grid, via arrays (many
cuts at strap intersections) are mandatory.

**Q17: What is the noise budget and how is it allocated?**

The total noise margin (~0.35V at VDD=0.7V) must accommodate all noise sources: crosstalk
coupling (~40%), power supply noise / IR drop (~30%), and process variation (~30%). Each
source is allocated a budget, and the design must keep each noise type within its budget.
If one source exceeds its allocation (e.g., severe IR drop), the margin for others shrinks,
potentially causing failures. Noise budget management is critical for low-voltage designs.

**Q18: What tools are used for reliability signoff?**

EM/IR: RedHawk-SC (Synopsys), Voltus (Cadence) — analyze power grid integrity, current
density, voltage drop. Aging: MOSRA in PrimeTime (Synopsys), aging models in Tempus
(Cadence) — BTI and HCI degradation analysis. ESD: commercial ESD checkers integrated
with DRC tools (Calibre, IC Validator). Thermal: ANSYS RedHawk-SC Electrothermal,
Voltus-Fi, Cadence Celsius — chip and package thermal analysis.

**Q19: How does temperature inversion affect reliability analysis?**

At low VDD (< 0.8V), the fastest corner shifts from high-temperature to low-temperature
(because Vth reduction at high T is more beneficial than mobility loss). This means hold
timing must be checked at low temperature (fast corner), not just high temperature. For
reliability, this complicates analysis because aging and EM models assume worst-case at
high temperature, but timing margins may be worse at low temperature.

**Q20: What is stress migration and how does it differ from EM?**

Stress migration (SM) is void formation in metal due to mechanical stress, not electrical
current. Thermal cycling creates stress between metal and dielectric (different thermal
expansion coefficients). Voids form at stress concentration points (via interfaces, line
ends). Unlike EM, SM occurs even without current flow — it can affect idle metal lines.
SM is checked separately from EM and has its own lifetime models based on stress gradient
and temperature cycling range.
