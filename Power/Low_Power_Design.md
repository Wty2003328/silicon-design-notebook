# Low Power Design -- The Complete Interview Bible

## Table of Contents
1. Switching Power -- First-Principles Derivation
2. Short-Circuit Power
3. Leakage Power -- MOSFET Physics Deep Dive
4. Clock Gating -- ICG Cell Design and Analysis
5. DVFS -- Voltage-Frequency Relationship
6. Power Gating -- Header/Footer Switch Design
7. Multi-Vt Optimization
8. Body Biasing (Forward and Reverse)
9. Practical Power Budgeting and Thermal Management
10. UPF Specification -- Production-Level Example
11. Power Analysis Methodology
12. Low-Power Design Flow
13. Interview Q&A (20+ Questions)

---

## 1. Switching Power -- First-Principles Derivation

### 1.1 Charging a Capacitor Through PMOS

Consider a CMOS inverter output transitioning from 0 to VDD (charging load capacitance CL through the PMOS pull-up).

```
        VDD
         |
       [PMOS]  <- turns ON for 0->1 transition
         |
    out--+--[CL]--GND
         |
       [NMOS]  <- turns OFF
         |
        GND
```

**Energy drawn from VDD during 0->1 transition:**

```
The current from VDD charges CL from 0 to VDD.

Charge delivered: Q = CL * VDD

Energy from supply: E_supply = Q * VDD = CL * VDD^2
```

**Energy stored in capacitor:**

```
E_cap = (1/2) * CL * VDD^2
```

**Energy dissipated in PMOS channel resistance:**

```
E_PMOS = E_supply - E_cap = CL * VDD^2 - (1/2) * CL * VDD^2 = (1/2) * CL * VDD^2
```

**Key insight:** Exactly HALF the energy from VDD is stored in CL, and HALF is dissipated as heat in the PMOS resistance. This is true regardless of PMOS size or channel resistance (assuming full swing).

### 1.2 Discharging Through NMOS (1->0 transition)

```
The stored energy (1/2) * CL * VDD^2 is dissipated in the NMOS channel.
No energy drawn from VDD (PMOS is off, current flows from CL to GND through NMOS).
```

### 1.3 Total Energy Per Full Cycle (0->1->0)

```
E_cycle = E_charge + E_discharge
        = (1/2) * CL * VDD^2 + (1/2) * CL * VDD^2
        = CL * VDD^2

Energy from VDD per cycle: CL * VDD^2  (all dissipated as heat)
```

### 1.4 Switching Power Formula

```
P_switching = alpha * CL * VDD^2 * f

where:
  alpha = activity factor = probability of 0->1 transition per clock cycle
  CL    = total load capacitance (gate + wire + fanout)
  VDD   = supply voltage
  f     = clock frequency

For a clock net (alpha = 1.0, toggles every half-cycle):
  P_clock = CL_total * VDD^2 * f

For typical logic (alpha ~ 0.1-0.3):
  P_logic = 0.1 to 0.3 * CL * VDD^2 * f
```

**Numerical example:**

```
Block with 1 million gates
Average CL per gate: 2 fF (including wire cap)
Total CL: 1M * 2 fF = 2 nF
Alpha: 0.15
VDD: 0.8V
Frequency: 1 GHz

P_switching = 0.15 * 2e-9 * (0.8)^2 * 1e9
            = 0.15 * 2e-9 * 0.64 * 1e9
            = 0.192 W = 192 mW
```

**Voltage scaling impact:**

```
Reducing VDD from 0.8V to 0.72V (10% reduction):
P_new = 0.15 * 2e-9 * (0.72)^2 * 1e9 = 0.15 * 2e-9 * 0.5184 * 1e9 = 155.5 mW

Savings: (192 - 155.5) / 192 = 19%  (quadratic!)

Reducing VDD from 0.8V to 0.6V (25% reduction):
P_new = 0.15 * 2e-9 * (0.36) * 1e9 = 108 mW
Savings: 44%  (but frequency must also drop significantly)
```

---

## 2. Short-Circuit Power

### 2.1 Mechanism

During an input transition, there is a brief period when both PMOS and NMOS are simultaneously conducting (input voltage is between Vtn and VDD - |Vtp|). A direct current path exists from VDD to GND.

```
      VDD
       |
     [PMOS] <- partially ON
       |
  Vin--+--Vout
       |
     [NMOS] <- partially ON
       |
      GND

Short-circuit window: Vtn < Vin < VDD - |Vtp|
```

### 2.2 Short-Circuit Power Estimate

```
P_sc = I_peak * (t_sc / T_clk) * VDD

where:
  I_peak = peak short-circuit current
  t_sc   = duration of short-circuit window during each transition
  T_clk  = clock period

Typical: P_sc = 5-15% of P_switching

Minimized when: input slew ~ output slew (balanced transition times)
```

When input slew is much slower than output slew, both transistors are ON for a long time, increasing P_sc. This is why excessive slew (failing max_transition DRC) increases power.

---

## 3. Leakage Power -- MOSFET Physics Deep Dive

### 3.1 Sub-Threshold Leakage (Dominant in Modern Nodes)

The MOSFET does not turn off abruptly at Vgs = Vth. Below threshold, current is dominated by diffusion (not drift):

```
I_sub = I_0 * exp((Vgs - Vth) / (n * Vt)) * (1 - exp(-Vds / Vt))

where:
  I_0  = technology-dependent constant (proportional to W/L)
  Vgs  = gate-to-source voltage (0 when "off" in CMOS)
  Vth  = threshold voltage
  n    = sub-threshold swing coefficient (1.0 to 1.5, ideally 1.0)
  Vt   = thermal voltage = kT/q = 26mV at 300K (27C)
  Vds  = drain-to-source voltage

For a "fully off" transistor (Vgs = 0, Vds = VDD >> Vt):
  I_sub = I_0 * exp(-Vth / (n * Vt))
```

### 3.2 Sub-Threshold Slope

The sub-threshold slope (SS) defines how many millivolts of Vgs change are needed to reduce current by 10x (one decade):

```
SS = n * Vt * ln(10) = n * 26mV * 2.303 = 60mV/decade (ideal at 300K, n=1)

Practical values: 60-100 mV/decade (n=1.0 to 1.7)
```

**Physical limit:** At room temperature, the absolute minimum SS is ~60 mV/decade. This is the **Boltzmann tyranny** -- no conventional MOSFET can switch faster than this. It fundamentally limits how much you can reduce Vth (and thus leakage) while maintaining a reasonable Ion/Ioff ratio.

```
For Vth = 0.3V and SS = 70 mV/decade:
  Decades off = Vth / SS = 0.3 / 0.07 = 4.3 decades
  Ion/Ioff = 10^4.3 = ~20,000:1

For Vth = 0.2V and SS = 70 mV/decade:
  Decades off = 0.2 / 0.07 = 2.86 decades
  Ion/Ioff = 10^2.86 = ~724:1   (marginal for logic!)
```

### 3.3 Temperature Dependence of Sub-Threshold Leakage

```
I_sub ~ exp(-Vth / (n * kT/q))

As T increases:
  1. Vt = kT/q increases linearly (26mV at 300K -> 34mV at 400K)
  2. Vth decreases (approximately -1 to -2 mV/K)
  3. Both effects increase leakage exponentially

Rule of thumb: leakage roughly DOUBLES for every 10-12C temperature rise
              (sometimes quoted as 10x per 30C for conservative estimates)
```

**Numerical example:**

```
Leakage at 25C (298K): 1 mA
At 85C (358K):
  Factor = exp((-Vth_hot)/n*Vt_hot) / exp((-Vth_cold)/n*Vt_cold)

  Using rule of thumb (2x per 10C):
  Factor = 2^((85-25)/10) = 2^6 = 64x
  Leakage at 85C ~ 64 mA

  Using conservative rule (10x per 30C):
  Factor = 10^((85-25)/30) = 10^2 = 100x
  Leakage at 85C ~ 100 mA

The actual value depends on the technology, but leakage at 125C can easily be
100-300x the value at 25C. This makes thermal management critical.
```

### 3.4 DIBL (Drain-Induced Barrier Lowering)

At short channel lengths, the drain voltage affects the source-channel barrier:

```
Vth_effective = Vth_0 - eta * Vds

where eta = DIBL coefficient (0.02 to 0.1 V/V for short channels)
```

When Vds increases (logic HIGH applied to drain), Vth decreases, increasing sub-threshold leakage. This is why stacking transistors (series NMOS) reduces leakage -- the intermediate node voltage reduces Vds across each transistor.

```
Single NMOS off (Vgs=0, Vds=VDD=0.8V):
  I_leak = I_0 * exp(-0.3 / (1.2 * 0.026)) = I_0 * exp(-9.62)

Stacked 2 NMOS off (each has Vds ~ VDD/2 = 0.4V due to intermediate node):
  Vth_eff = 0.3 + 0.05*0.4 = 0.32V (DIBL correction, Vds reduced)
  I_leak ~ I_0 * exp(-0.32 / (1.2 * 0.026)) = I_0 * exp(-10.26)

Ratio: exp(-9.62) / exp(-10.26) = exp(0.64) = 1.9x reduction

The "stack effect" provides ~2-3x leakage reduction for 2-high stacks.
```

### 3.5 Gate Oxide Tunneling Leakage

Electrons can quantum-mechanically tunnel through the thin gate oxide.

**Direct tunneling** (oxide < 3nm): Dominates at advanced nodes.

```
I_gate ~ A * (Vox/Tox)^2 * exp(-B * Tox * sqrt(phi_b - Vox/2))

where:
  Tox  = oxide thickness
  Vox  = voltage across oxide
  phi_b = barrier height (~3.1 eV for SiO2)
  A, B = constants
```

Gate leakage is exponentially sensitive to oxide thickness. Each 0.1nm reduction roughly 10x increases tunneling current.

**High-k dielectrics (HfO2, etc.):**

```
Physical thickness can be larger (reducing tunneling) while maintaining
the same effective capacitance:

  C = epsilon_0 * k / T_physical

SiO2:  k = 3.9, T = 1.2nm -> C = epsilon_0 * 3.9 / 1.2nm
HfO2:  k = 25,  T = 5nm   -> C = epsilon_0 * 25 / 5nm = same C!

But tunneling through 5nm is negligible compared to 1.2nm.
High-k reduced gate leakage by 100-1000x at 45nm and below.
```

**Equivalent Oxide Thickness (EOT):**

```
EOT = T_physical * (k_SiO2 / k_high-k) = 5nm * (3.9/25) = 0.78nm

This means the HfO2 gate has the capacitance of a 0.78nm SiO2 gate
but the tunneling of a 5nm barrier.
```

### 3.6 GIDL (Gate-Induced Drain Leakage)

When the gate is at 0V and the drain is at VDD, a strong electric field at the gate-drain overlap causes band-to-band tunneling:

```
          Gate (0V)
     ═══════════════
          |
     ─────+───── Drain (VDD)
     |         |
  Valence    Conduction
   band       band
     |         |
     └── e-h+ ─┘  (tunneling across bandgap)
```

GIDL is significant when |Vgd| > ~1V and becomes worse with thinner oxides and higher drain doping. It's a concern for DRAM retention (off-transistor leaking charge from storage capacitor) and for FinFET devices where the gate wraps around the fin, increasing the overlap field.

### 3.7 Leakage Power Breakdown by Node

```
| Source          | 65nm  | 28nm  | 7nm FinFET |
|-----------------|-------|-------|------------|
| Sub-threshold   | 70%   | 80%   | 85%        |
| Gate tunneling  | 20%   | 10%   | 5%         |
| Junction/GIDL   | 10%   | 10%   | 10%        |

Note: FinFET dramatically reduced gate leakage (thicker effective oxide
due to the fin geometry) and improved sub-threshold slope (n closer to 1.0),
but leakage still dominates due to the sheer number of transistors.
```

---

## 4. Clock Gating -- ICG Cell Design and Analysis

### 4.1 ICG Cell Schematic (Latch + AND Gate)

```
               ┌─────────────────────┐
   EN ────────>│ D    Latch    Q ├───┐
               │         CLK        │   │
               │          |         │   │
   CLK ───────>│──────────┘         │   ├──[AND]──> GCLK
               │                    │   │     |
               └────────────────────┘   └─────┘
                                              |
                                             CLK
```

**Why latch-based is glitch-free (timing diagram):**

```
CLK:     ___|‾‾‾‾‾‾|______|‾‾‾‾‾‾|______|‾‾‾‾‾‾|______
EN:      __|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_________________________
EN_latch:_____|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|__________________
GCLK:   ______|‾‾‾‾‾‾|______|‾‾‾‾‾‾|_______________________

Latch is transparent when CLK=0 (LOW phase).
EN changes during CLK=0 are captured by the latch.
EN changes during CLK=1 are BLOCKED by the latch (it's opaque).
The AND gate only sees EN_latched, which is stable during CLK=1.
Therefore, GCLK = CLK & EN_latched is GLITCH-FREE.

Without the latch:
CLK:     ___|‾‾‾‾‾‾|______|‾‾‾‾‾‾|______
EN:      ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_______
GCLK:   _______|‾‾‾‾|______|‾‾‾‾‾‾|_____
                ^^ GLITCH! EN arrived while CLK was high
                   causing a partial pulse on GCLK
```

### 4.2 ICG Timing Constraints in STA

The ICG has its own setup and hold checks:

```
EN must meet SETUP before the falling edge of CLK (latch opening edge):
  EN arrival time < CLK_fall - T_setup_icg

EN must meet HOLD after the falling edge of CLK:
  EN arrival time > CLK_fall + T_hold_icg
```

These appear as "clock gating check" timing reports in PrimeTime.

### 4.3 Power Savings Calculation

**Example: 1000 flip-flops with clock gating, gated 70% of the time**

```
Assumptions:
  Each FF clock pin capacitance: 5 fF
  Clock tree buffer capacitance per FF: 3 fF (average)
  ICG cell capacitance: 15 fF
  VDD = 0.8V, f = 1 GHz
  Activity without gating: alpha = 1.0 (clock toggles every cycle)

Power WITHOUT gating:
  Total clock cap = 1000 * (5 + 3) fF = 8000 fF = 8 pF
  P_clock = 1.0 * 8e-12 * (0.8)^2 * 1e9 = 5.12 mW

Power WITH gating (gated 70%, active 30%):
  ICG cell always toggles: P_icg = 1.0 * 15e-15 * 0.64 * 1e9 = 0.0096 mW
  Gated tree (active 30%):  P_gated = 0.3 * 8e-12 * 0.64 * 1e9 = 1.536 mW
  Total = 0.0096 + 1.536 = 1.546 mW

Savings = 5.12 - 1.546 = 3.574 mW = 70% savings!
```

### 4.4 AND-Type vs OR-Type ICG Cells

```
AND-type ICG (clock gating for active-high enables, INDUSTRY STANDARD):
  
  GCLK = CLK & EN_latched
  
  The latch captures EN on the NEGATIVE phase (transparent when CLK=0).
  During CLK=1: latch is opaque, EN_latched is stable.
  AND gate: GCLK is HIGH only when both CLK=1 AND EN_latched=1.
  
  Use when: gating signal is active-HIGH enable (1 = clock enabled).
  This is the default for 95%+ of clock gating in digital designs.
  When EN=0: GCLK is stuck LOW -> flip-flops see no rising edge.

OR-type ICG (clock gating for active-low enables):
  
  GCLK = CLK | EN_latched_n
  
  The latch captures EN on the POSITIVE phase (transparent when CLK=1).
  During CLK=0: latch is opaque, EN_latched is stable.
  OR gate: GCLK is LOW only when both CLK=0 AND EN_latched_n=0.
  
  Use when: gating signal is active-LOW (0 = clock enabled).
  When EN=1: GCLK is stuck HIGH -> flip-flops see no rising edge.
  
  Rare in practice -- most designs use AND-type with inverted enable if needed.

When to use each:
  - AND-type: standard register file gating, pipeline stage gating, module gating
  - OR-type: rarely needed; some negative-edge-triggered designs use it
  - Most synthesis tools only insert AND-type ICG cells

Timing of enable relative to clock:
  AND-type: EN must be stable by FALLING edge of CLK (setup to negedge)
  OR-type:  EN must be stable by RISING edge of CLK (setup to posedge)
  
  The setup requirement is to the INACTIVE edge, not the active edge.
  This gives a full half-cycle for enable to propagate from launching FF.
```

### 4.4B Clock Gating Enable Generation from RTL

```
The synthesis tool infers ICG from these RTL patterns:

Pattern 1: if-else with clocked assignment (MOST COMMON)
  always @(posedge clk)
    if (enable)           // <-- ICG inferred here
      q <= d;

  Tool actions:
    1. Extracts "enable" as the gating signal
    2. Inserts ICG cell on the clock path
    3. Removes the mux on D input (saves combinational logic)
    
  Without clock gating: tool implements D-input mux
    always @(posedge clk)
      q <= enable ? d : q;  // mux on D, clock always toggles

Pattern 2: Explicit clock gating (manual)
  wire gated_clk;
  ICG_CELL u_icg (.CLK(clk), .EN(enable), .GCLK(gated_clk));
  always @(posedge gated_clk)
    q <= d;

Pattern 3: Module-level gating (coarse-grain)
  // In the power management controller:
  always @(posedge clk) begin
    if (module_active)
      module_clk_en <= 1;
    else
      module_clk_en <= 0;
  end
  // ICG for entire module inserted at clock entry point

Module-level vs gate-level clock gating:
  Module-level: one ICG for entire module (1 cell for 10K FFs)
    - Controlled by software/firmware
    - Gating efficiency depends on module idle time
    - Very low area overhead
  
  Gate-level (RTL-inferred): one ICG per 4-64 FFs
    - Controlled by data path enables
    - Gating efficiency depends on data-dependent activity
    - Higher area overhead (many ICG cells)
    - Typical: 5-8% area increase, 20-40% power reduction
  
  Best practice: use BOTH. Module-level for sleep modes,
  gate-level for dynamic activity reduction during active operation.
```

### 4.4C Fine-Grain vs Coarse-Grain Clock Gating

| Aspect | Fine-Grain | Coarse-Grain |
|--------|-----------|--------------|
| Granularity | 4-16 FFs per ICG | 100-1000+ FFs per ICG |
| Area overhead | High (many ICGs) | Low (few ICGs) |
| Gating efficiency | High (precise control) | Lower (all-or-nothing) |
| Typical source | Synthesis auto-insertion | Architecture/RTL manual |
| Example | Register enable per field | Module-level sleep |

**Synthesis auto-insertion threshold:**

```tcl
# In Design Compiler:
set_clock_gating_style -minimum_bitwidth 4  ; # Gate groups of 4+ FFs
set_clock_gating_style -max_fanout 64       ; # Max FFs per ICG
compile_ultra -gate_clock
```

The tool identifies registers with a common enable signal and inserts ICG cells. The enable is factored out of the D-input logic and connected to the ICG.

---

## 5. DVFS -- Voltage-Frequency Relationship

### 5.1 Gate Delay vs Voltage

The propagation delay of a CMOS gate is:

```
T_pd = (CL * VDD) / I_avg

I_avg ~ (W/L) * mu * Cox * (VDD - Vth)^alpha / 2

Therefore:
  T_pd ~ CL * VDD / ((VDD - Vth)^alpha)

where alpha is between 1 (velocity-saturated) and 2 (long-channel square-law)
For modern short-channel: alpha ~ 1.0 to 1.3
```

**Maximum frequency:**

```
f_max = 1 / T_pd ~ (VDD - Vth)^alpha / (CL * VDD)

For VDD >> Vth: f_max ~ VDD^(alpha-1) ~ roughly proportional to VDD
For VDD near Vth: f_max drops rapidly (approaching 0 at VDD = Vth)
```

**DVFS power scaling:**

```
P_dynamic = alpha_sw * CL * VDD^2 * f

If we reduce VDD by factor k (VDD_new = k * VDD_old):
  f_new ~ k * f_old  (approximate, for VDD >> Vth)
  P_new = alpha * CL * (k*VDD)^2 * (k*f) = k^3 * P_old

Cubic power reduction! A 20% voltage reduction gives:
  P_new = 0.8^3 * P_old = 0.512 * P_old = 49% savings
```

### 5.2 Practical DVFS Implementation

```
                    ┌──────────────┐
   Performance  -->│  DVFS        │--> VDD_request --> [Voltage Regulator]
   Request         │  Controller  │--> Freq_request --> [PLL/Clock Divider]
                    │              │
   Voltage_good <--│              │<-- VDD_actual
   PLL_locked   <--│              │<-- CLK_actual
                    └──────────────┘

Voltage UP sequence:
  1. Increase VDD first (takes 1-50us for regulator settling)
  2. Wait for voltage to stabilize (monitor VDD_good)
  3. THEN increase frequency (PLL relock: 1-10us)

Voltage DOWN sequence:
  1. Decrease frequency first (immediate or PLL relock)
  2. THEN decrease VDD
  3. Wait for voltage to settle

WHY this order matters:
  - If you increase frequency before voltage: gates may not switch fast enough
    at the old (low) voltage -> timing violations -> functional failure
  - If you decrease voltage before frequency: same problem, gates too slow
    for the current (high) frequency
```

### 5.2B Voltage-Frequency Curve Derivation

For a gate with delay T_d proportional to Vdd / (Vdd - Vth)^alpha:

```
For constant delay (same performance): Vdd must scale with frequency
  If f doubles: T_d halves -> need Vdd to increase
  Roughly: Vdd proportional to f for constant delay
  (at Vdd >> Vth: T_d ~ Vdd / Vdd^alpha = 1/Vdd^(alpha-1), 
   so Vdd ~ f^(1/(alpha-1)))

For alpha = 1.3: Vdd ~ f^(1/0.3) = f^3.33
  If f increases 20%: Vdd must increase ~80%
  This is why small frequency increases cost enormous power!
```

**Detailed operating points table with derivation:**

```
Assume: 7nm FinFET, Vth = 0.25V, alpha = 1.3
Base design at 0.75V, 2.0 GHz

| OPP     | Vdd(V) | F(GHz) | Vdd-Vth | Relative T_d | Relative P_dyn | Use Case         |
|---------|--------|--------|---------|-------------|----------------|-------------------|
| Turbo   | 0.95   | 2.8    | 0.70    | 0.72x (fast)| 1.78x          | Peak burst        |
| Nominal | 0.80   | 2.0    | 0.55    | 1.00x (ref) | 1.00x          | Sustained         |
| SVS     | 0.70   | 1.4    | 0.45    | 1.33x       | 0.64x          | Light load        |
| Low     | 0.60   | 0.8    | 0.35    | 1.90x       | 0.34x          | Background        |
| Min     | 0.50   | 0.4    | 0.25    | 2.98x       | 0.14x          | Always-on sensor  |

Relative P_dyn = (Vdd/Vdd_ref)^2 * (f/f_ref)
  Turbo:  (0.95/0.80)^2 * (2.8/2.0) = 1.41 * 1.40 = 1.97 (close to 1.78 with rounding)
  SVS:    (0.70/0.80)^2 * (1.4/2.0) = 0.766 * 0.70  = 0.54
  Low:    (0.60/0.80)^2 * (0.8/2.0) = 0.563 * 0.40  = 0.23
  Min:    (0.50/0.80)^2 * (0.4/2.0) = 0.391 * 0.20  = 0.08

Power at Turbo = 1.97x the nominal -> thermal throttling kicks in within seconds.
Power at Min   = 0.08x nominal -> 92% dynamic power reduction for background tasks.
```

### 5.2C DVFS Controller Decision Logic

```
How the DVFS controller decides V/F level:

Method 1: Utilization-based (most common in mobile SoCs)
  1. Hardware performance counters track:
     - Instructions per cycle (IPC)
     - Cache miss rate
     - Pipeline stall cycles
     - Memory bandwidth utilization
  2. OS governor samples counters every 10-100 ms
  3. Decision algorithm:
     utilization = busy_cycles / total_cycles
     
     if utilization > 80%: scale UP to next OPP
     if utilization < 30%: scale DOWN to lower OPP
     if 30% < utilization < 80%: maintain current OPP
  4. Hysteresis: require N consecutive samples above/below threshold
     to prevent oscillation (typically N=3 samples at 50ms each = 150ms)

Method 2: Workload prediction (used in modern CPUs)
  - Track workload history (last 10-100 ms)
  - Predict next interval's demand using moving average
  - Preemptively adjust V/F before the workload arrives
  - Reduces response latency for bursty workloads

Method 3: Critical path monitor (hardware AVS)
  - On-die ring oscillators or replica delay lines
  - Measure actual silicon speed in real-time
  - If replica path is faster than needed -> reduce Vdd
  - If replica path is too slow -> increase Vdd
  - Per-chip optimization: fast silicon saves 50-100 mV
```

### 5.2D Voltage Regulator Response Time

```
How fast can you change Vdd?

External PMIC (Power Management IC):
  - Buck converter on the PCB
  - Voltage change: 10-50 us settling time
  - Step size: 6.25 mV or 12.5 mV resolution
  - Limited by output inductor and capacitor
  - Most mobile SoCs use this (external PMIC)

On-die LDO (Low Dropout Regulator):
  - Analog LDO on the same die as the logic
  - Voltage change: 0.5-5 us settling time
  - Much faster than external PMIC
  - But LDO efficiency = Vout/Vin (at 0.6V out / 1.0V in = 60% efficient)
  - Used for fine-grain DVFS with fast transitions

DVFS transition overhead:
  Scale up:   10-50 us (wait for PMIC) + 5-20 us (PLL relock) = 15-70 us total
  Scale down: 5-20 us (PLL relock) + 10-50 us (voltage settles) = 15-70 us total
  
  During transition: logic continues at old V/F (safe but suboptimal)
  
  Energy wasted during transition:
    E_transition = P_old * T_transition / 2 (roughly, average of old and new)
    For a 50 us transition at 1W average: E = 25 uJ
    
  Minimum DVFS interval: transition cost must be amortized
    If saving 500 mW by switching down for T_idle:
    Break-even: 500mW * T_idle > 25 uJ -> T_idle > 50 us
    Practical minimum: 1-10 ms between DVFS transitions
```

### 5.3 DVFS Operating Points -- Real-World Example

```
| OPP     | VDD (V) | Freq (GHz) | Rel Power | Use Case          |
|---------|---------|------------|-----------|-------------------|
| Turbo   | 0.95    | 2.5        | 1.80      | Peak performance  |
| Nominal | 0.80    | 2.0        | 1.00      | Normal operation  |
| Low     | 0.72    | 1.5        | 0.55      | Light workload    |
| Ultra-low| 0.60   | 0.8        | 0.22      | Always-on sensor  |
| Retention| 0.50   | 0 (stopped)| 0.08      | Deep sleep (leak) |
```

### 5.4 Sub-Threshold Operation (VDD < Vth)

Below Vth, transistors still conduct via diffusion current, but:

```
- Current drops exponentially: I ~ exp(VDD/Vt) for VDD < Vth
- Frequency drops to kHz-MHz range
- Ion/Ioff ratio approaches 1 -> poor noise margins
- Very sensitive to Vth variation (process corners spread widely)
- Used for ultra-low-power sensor nodes, pacemakers

At VDD = 0.3V (sub-Vth for Vth=0.4V):
  Freq ~ 10-100 kHz (application-specific)
  Power ~ 1-10 uW (nW-range for simple logic)
  Energy per operation: LOWER than super-threshold! (E ~ CV^2 is quadratic)
```

---

## 6. Power Gating -- Deep Dive

### 6.1 Header vs Footer Switches

**Header switch (PMOS between VDD and virtual VDD):**

```
     Real VDD
       |
    [PMOS header]  <- SLEEP_N controls
       |
    Virtual VDD (VVDD)
       |
    [Logic block]
       |
      GND
```

**Footer switch (NMOS between virtual GND and real GND):**

```
    VDD
     |
  [Logic block]
     |
  Virtual GND (VGND)
     |
  [NMOS footer]  <- SLEEP controls
     |
    Real GND
```

**Comparison:**

| Aspect | Header (PMOS) | Footer (NMOS) |
|--------|--------------|---------------|
| Drive strength | Weaker (PMOS mobility lower) | Stronger (NMOS) |
| Switch size | ~2x larger for same resistance | 1x (baseline) |
| Voltage drop | IR drop on VDD | IR drop on GND |
| Ground bounce | None (GND is solid) | Can cause noise |
| Layout | VDD rail has switches | GND rail has switches |
| Typical use | Most common in practice | Used when GND mesh is easier |

**Header switches are more common** because GND bounce from footer switches can cause functional errors in always-on logic sharing the same GND mesh.

### 6.2 Rush Current Calculation and Switch Sizing

When power is restored, the virtual rail must charge from 0V to VDD. The parasitic capacitance of the entire gated domain (gate caps + wire caps + decoupling caps) creates a rush current.

```
I_rush = C_total * dV/dt

Example:
  Domain area: 1 mm^2
  Decoupling cap: ~10 nF/mm^2 (typical for dense logic)
  C_total = 10 nF
  Target ramp time: 10 us (controlled ramp to limit rush current)

  I_rush_max = C_total * VDD / T_ramp = 10e-9 * 0.8 / 10e-6 = 0.8 mA
  (This is very manageable)

  If uncontrolled (T_ramp ~ 1 ns -- switch turns on instantly):
  I_rush = 10e-9 * 0.8 / 1e-9 = 8A !!
  (This would cause massive IR drop, EM violations, and ground bounce)
```

### 6.3 Switch Sizing

The header/footer switch must be sized to:

1. **Have low enough resistance in ON state** to limit IR drop during normal operation.
2. **Not be too large** (area and leakage overhead).

```
Switch ON resistance:
  R_switch = Leff / (W_total * mu * Cox * (VDD - |Vtp|))

Target: R_switch * I_max < IR_drop_budget

Example:
  I_max (peak current of domain): 100 mA
  IR_drop budget: 5% of VDD = 40 mV
  Required R_switch < 40mV / 100mA = 0.4 ohm

  For PMOS header in 7nm:
    R_per_fin ~ 500 ohm (per fin, Leff = 7nm)
    W_total needed = R_per_fin / R_switch = 500/0.4 = 1250 fins
    At 100 fins per header cell, need ~13 header cells

  These cells are distributed along the VDD rail (not concentrated in one spot).
```

### 6.4 Daisy-Chain Power-On (Rush Current Control)

```
   SLEEP_N[0] --> [Header_group_0] --> ack_0
                                        |
   SLEEP_N[1] <---- (ack_0 delayed) <---+
                  --> [Header_group_1] --> ack_1
                                           |
   SLEEP_N[2] <---- (ack_1 delayed) <------+
                  --> [Header_group_2] --> ack_2
                  ...

Turn on header groups sequentially with delay between each group.
Each group charges a portion of the capacitance.
Rush current spread over time -> manageable peak current.
```

### 6.5 Complete Power Gating Sequence

```
Power-DOWN sequence:
  1. Software saves context (optional, for state not in retention FFs)
  2. Assert SAVE signal -> retention FFs latch their values
  3. Assert ISOLATION -> outputs clamped to known values
  4. Assert SLEEP -> header switches turn OFF
  5. Virtual VDD drops to 0V (domain is dead)
  6. Only retention FFs (on always-on supply) hold state

Power-UP sequence:
  1. Deassert SLEEP -> header switches turn ON (daisy-chain)
  2. Wait for VVDD to stabilize (ramp time + settling)
  3. Assert RESET to all FFs (initialize state machines)
  4. Deassert RESET (synchronized to clock!)
  5. Deassert ISOLATION -> outputs driven by live logic
  6. Assert RESTORE -> retention FFs restore saved values
  7. Deassert RESTORE
  8. Domain is operational

Critical ordering rules:
  - ISOLATION must be asserted BEFORE power-off (else floating outputs corrupt always-on domain)
  - ISOLATION must be deasserted AFTER power is stable and reset is done
  - SAVE must happen BEFORE power-off
  - RESTORE must happen AFTER power is stable
```

### 6.6 Retention Register Design (Balloon Latch)

```
         Main FF (on switchable supply VVDD)
  ┌──────────────────────────────┐
  │  D --[TG]--M--[TG]--S-- Q   │
  │         |         |          │
  │       [INV]     [INV]       │
  │         |         |          │
  └──────────┬───────────────────┘
             │
          [Shadow Latch]  (on ALWAYS-ON supply VDD)
          ┌──────────────┐
          │  M --[TG]--R │
          │       |      │
          │     [INV]    │
          │       |      │
          └──────────────┘

  SAVE signal:  TG opens, M value stored in R (shadow)
  RESTORE signal: TG opens, R value driven back to M

  Shadow latch power: always-on (tiny: 2 transistors + feedback inverter)
  Area overhead: ~30-50% larger than normal FF
  Leakage overhead: shadow latch leakage on always-on rail
```

### 6.6B Retention Register Detailed Design (Balloon Register)

```
Schematic of a retention flip-flop with balloon (shadow) latch:

  Switchable VDD (VVDD)                    Always-On VDD (VDD_AO)
  ┌────────────────────────────┐           ┌──────────────────────┐
  │                            │           │                      │
  │  D ──[TG1]── Master ──[TG2]── Slave ──│── Q                  │
  │       ↑        Latch       ↑   Latch   │                      │
  │      CLK1                 CLK2         │                      │
  │                            │           │   Shadow Latch       │
  │                      [TG3]─┤───────────│──[TG4]── Retain ─────│
  │                       ↑    │           │   ↑     Latch        │
  │                     SAVE   │           │ RESTORE               │
  └────────────────────────────┘           └──────────────────────┘
  
  TG1/TG2: Transmission gates (clocked by CLK, powered by VVDD)
  TG3:     SAVE transmission gate (powered by VVDD for input, VDD_AO for output)
  TG4:     RESTORE transmission gate (powered by VDD_AO)
  
  Normal operation (VVDD = ON):
    Master-slave operates as normal FF
    Shadow latch is idle (SAVE=0, RESTORE=0)
    
  SAVE sequence (before power-down):
    1. Clock is stopped (no active edge)
    2. SAVE is pulsed HIGH: TG3 opens, Slave output -> Shadow input
    3. Shadow latch captures the bit on VDD_AO supply
    4. SAVE goes LOW: TG3 closes, Shadow holds the bit
    
  Power-down:
    VVDD ramps to 0V. Master and Slave lose all state.
    Shadow latch retains on VDD_AO. Only ~4-6 transistors draw leakage.
    
  RESTORE sequence (after power-up):
    1. VVDD is stable at target voltage
    2. Master/Slave are in unknown state (random)
    3. RESTORE is pulsed HIGH: TG4 opens, Shadow output -> Master input
    4. Master captures the saved bit from Shadow
    5. RESTORE goes LOW
    6. Next clock edge: Slave captures from Master -> Q has saved value
    
Area overhead: ~30-50% larger than standard FF
  Standard D-FF: ~16-20 transistors
  Retention D-FF: ~26-34 transistors (shadow latch + TG3/TG4 + control logic)
  
Leakage overhead: only the shadow latch leaks on the always-on rail
  Shadow latch: ~4-6 transistors, ~2-5 nA at 25C
  For 10K retention FFs: 10,000 * 5nA * 0.9V = 45 uW always-on leakage
  This is the cost of fast wake-up (vs. full re-initialization)
```

### 6.6C Power-Up Sequence with In-Rush Current Management

```
Detailed power-up sequence with timing:

  t0: Wake-up event (interrupt, timer)
  t1: PMU asserts SLEEP_N deassertion (start turning on header switches)
      - Daisy-chain: group 0 turns on first
      - Each group delayed by ~200 ns from the previous
      - With 10 groups: total ramp time = 10 * 200 ns = 2 us
      - In-rush current limited to: I_domain / N_groups
        = 100 mA / 10 = 10 mA per group (manageable)
        
      Without daisy-chain:
        I_rush = C_domain * VDD / T_ramp_uncontrolled
               = 10 nF * 0.8V / 1 ns (instantaneous switch)
               = 8A !! (catastrophic Ldi/dt, package bondwire fuse risk)
        
  t2: VVDD reaches 90% of VDD (after ~2 us)
      Voltage detector signals "power good"
      
  t3: PMU asserts RESET to domain (1 us minimum pulse width)
      All FFs go to known state (0)
      
  t4: PMU asserts RESTORE to retention FFs (1 us minimum pulse width)
      Shadow latches write back to master/slave
      
  t5: PMU deasserts RESET (synchronous to clock!)
      Domain FFs now have restored values
      
  t6: PMU deasserts ISOLATION
      Domain outputs go from clamped values to live logic
      
  t7: PMU enables clocks (via clock gating deassertion)
      First functional clock edge
      
  t8: Domain is fully operational
      Total wake-up time: ~5-15 us (domain dependent)

Critical ordering violations and their consequences:
  ISOLATION deasserted before VVDD stable:
    -> Outputs are undefined (random) -> corrupt always-on domain
  RESTORE before VVDD stable:
    -> Shadow latch data transfers incorrectly -> silent data corruption
  Clock enabled before RESET deasserted:
    -> FFs clock in garbage data for 1+ cycles
  RESET deasserted asynchronously:
    -> Recovery/removal violation on FFs -> metastability
```

### 6.7 Isolation Cells

| Type | Output When Isolated | Use Case |
|------|---------------------|----------|
| Clamp-to-0 | Output = 0 | Default for most signals |
| Clamp-to-1 | Output = 1 | Active-low signals, resets |
| Latch | Output = last value | When downstream needs stable data |
| High-Z | Output = Z | Shared bus, tristatable outputs |

```
Isolation cell (clamp-to-0):

  IN ──[AND]── OUT
         |
        ISO_N  (active-low: when ISO_N=0, output is forced to 0)

The AND gate is powered by the ALWAYS-ON supply (destination domain).
When the source domain is off, IN is floating, but ISO_N=0 forces OUT=0.
```

### 6.8 UPF for Power-Gated Domain

```tcl
# Define power domains
create_power_domain PD_TOP -include_scope
create_power_domain PD_GPU -elements {u_gpu}

# Supply network
create_supply_net VDD -domain PD_TOP
create_supply_net VDD_GPU -domain PD_GPU
create_supply_net VSS

create_supply_set PD_TOP.primary -function {power VDD} -function {ground VSS}
create_supply_set PD_GPU.primary -function {power VDD_GPU} -function {ground VSS}

# Power switch (header)
create_power_switch gpu_sw \
    -domain PD_GPU \
    -input_supply_port {vdd_in VDD} \
    -output_supply_port {vdd_out VDD_GPU} \
    -control_port {sleep_n u_pmu/gpu_sleep_n} \
    -on_state {on_state vdd_in {!sleep_n}}

# Isolation strategy
set_isolation iso_gpu \
    -domain PD_GPU \
    -isolation_power_net VDD \
    -isolation_ground_net VSS \
    -clamp_value 0 \
    -applies_to outputs \
    -isolation_signal {u_pmu/gpu_iso_en} \
    -isolation_sense high \
    -name_prefix ISO_

# Retention strategy
set_retention ret_gpu \
    -domain PD_GPU \
    -retention_power_net VDD \
    -retention_ground_net VSS \
    -save_signal {u_pmu/gpu_save HIGH} \
    -restore_signal {u_pmu/gpu_restore HIGH}

# Power states
add_power_state PD_GPU \
    -state GPU_ON  {-supply_expr {power == FULL_ON}} \
    -state GPU_OFF {-supply_expr {power == OFF}} \
    -state GPU_RET {-supply_expr {power == OFF, retention == FULL_ON}}
```

---

## 7. Multi-Vt Optimization

### 7.1 Actual Library Data (Typical 7nm FinFET)

```
| Cell     | Vt Type | Delay (ps) | Leakage (nW) | Rel Delay | Rel Leak |
|----------|---------|-----------|-------------|-----------|----------|
| INV_X1   | uLVT   | 12        | 150         | 0.80      | 15.0     |
| INV_X1   | LVT    | 14        | 50          | 0.93      | 5.0      |
| INV_X1   | SVT    | 15        | 10          | 1.00      | 1.0      |
| INV_X1   | HVT    | 19        | 2           | 1.27      | 0.2      |
| INV_X1   | uHVT   | 25        | 0.5         | 1.67      | 0.05     |

Observations:
- LVT is ~7% faster than SVT but 5x more leaky
- HVT is ~27% slower than SVT but 5x less leaky
- uLVT is ~20% faster than SVT but 15x more leaky
- The delay penalty is modest; the leakage difference is enormous
```

### 7.2 Vt Assignment Algorithm

```
Optimization strategy (typical in Synopsys Design Compiler / ICC2):

Phase 1: Start with ALL cells as HVT (minimum leakage)
  - Run STA, identify violating paths

Phase 2: Swap cells on critical paths from HVT -> SVT
  - Re-run STA, check if violations clear
  - If not enough, swap to LVT

Phase 3: Iterative refinement
  - For each remaining violation, find the cell on the critical path
    with the best delay-improvement-per-leakage-cost ratio
  - Swap that cell to a lower Vt
  - Repeat until timing is clean

Phase 4: Post-route leakage recovery
  - For each non-critical path, try swapping LVT -> SVT -> HVT
  - If setup slack remains positive, keep the swap (save leakage)
  - Process paths from most slack to least slack

Result: Typically 70-85% HVT, 10-20% SVT, 5-10% LVT
```

### 7.2B Multi-Vt Synthesis Flow in Detail

```
Step-by-step multi-Vt optimization in Synopsys Design Compiler:

Phase 0: Library preparation
  - Foundry provides 4-6 Vt flavors per cell (ULVT, LVT, SVT, HVT, UHVT)
  - Each has same function, different Vth implant / work function metal
  - Liberty files: nangate_7nm_UHVT.lib, nangate_7nm_HVT.lib, etc.
  
Phase 1: Initial compile (all HVT for minimum leakage)
  set_link_library "* 7nm_hvt.db 7nm_svt.db 7nm_lvt.db 7nm_ulvt.db"
  compile_ultra -scan
  
Phase 2: Vt assignment (swap up for timing closure)
  optimize_netlist -area -power
  # DC swaps cells from HVT -> SVT -> LVT on critical paths
  # Each swap: ~15-25% faster, ~4-5x more leakage
  
Phase 3: Leakage recovery (swap down for non-critical paths)
  # After timing closure, find cells with positive slack
  # Swap LVT -> SVT -> HVT if timing still meets
  # Priority: swap the highest-leakage cells first
  
Phase 4: Constraint-driven Vt limits
  set_multi_vth_constraint -lvth_percentage 10 -type soft
  # Limit LVT+ULVT to 10% of total cell count
  # Soft constraint: may be violated if timing requires
  
Phase 5: Report and verify
  report_threshold_voltage_group
  report_power -breakdown {vth}
  
Typical result for a well-optimized CPU core at 7nm:
  UHVT:  1%  (absolute critical paths only)
  HVT:  60%  (vast majority of non-critical logic)
  SVT:  28%  (moderately timing-critical)
  LVT:  10%  (near-critical paths)
  ULVT:  1%  (rarely needed, only for the most critical gates)
  
  Leakage breakdown by Vt:
    1% ULVT -> 25% of total leakage (each cell leaks 20-80x more than HVT!)
    10% LVT -> 35% of total leakage
    28% SVT -> 25% of total leakage  
    60% HVT -> 15% of total leakage
    
  This shows the Pareto: a few LVT/ULVT cells dominate total leakage.
  Aggressive leakage recovery on even 1% of LVT cells can save 5-10% total power.
```

### 7.2C Multi-Vt Leakage Reduction Quantified

```
Worked example: 5M gate design, 7nm FinFET

All-SVT baseline:
  Leakage per SVT gate = 10 nA (at 25C)
  Total leakage = 5e6 * 10e-9 * 0.75V = 37.5 mW

After multi-Vt optimization (60% HVT, 28% SVT, 10% LVT, 2% ULVT):
  HVT leakage:  3.0M * 10nA * 0.20x = 6.0 mA * 0.75 = 4.5 mW
  SVT leakage:  1.4M * 10nA * 1.00x = 14.0 mA * 0.75 = 10.5 mW
  LVT leakage:  0.5M * 10nA * 5.00x = 25.0 mA * 0.75 = 18.8 mW
  ULVT leakage: 0.1M * 10nA * 20.0x = 20.0 mA * 0.75 = 15.0 mW
  Total: 65 mA * 0.75V = 48.8 mW

Wait -- that's MORE than all-SVT! Why?

The issue: LVT/ULVT cells on critical paths add enormous leakage.
But these cells were NECESSARY to meet timing -- without them, the
design would need to run at lower frequency or higher voltage.

Compare at constant performance:
  All-SVT at 0.85V (higher voltage to meet timing):
    Dynamic: 5e6 * 0.15 * 1.2fF * 0.85^2 * 2GHz = 1.3 W
    Leakage:  37.5 mW (at 0.85V, slightly less due to lower Vds DIBL)
    Total: ~1.34 W
    
  Multi-Vt at 0.75V (lower voltage thanks to LVT on critical paths):
    Dynamic: 5e6 * 0.15 * 1.2fF * 0.75^2 * 2GHz = 1.01 W
    Leakage: 48.8 mW
    Total: ~1.06 W
    
  Net savings: 1.34 - 1.06 = 280 mW (21% reduction!)
  
The key insight: multi-Vt enables lower Vdd operation by speeding up
critical paths with LVT, which saves dynamic power quadratically.
The leakage increase from LVT is small compared to V^2 dynamic savings.

Typical industry result: multi-Vt optimization saves 2-5x total leakage
vs an all-LVT design (which would meet timing trivially but leak terribly),
and 20-30% total power vs all-SVT at higher voltage.
```

### 7.3 Vt vs Cell Sizing Trade-off

```
To fix a 20ps setup violation, you can:

Option A: Swap bottleneck cell from HVT to LVT
  Delay improvement: ~20%  (~10-15ps for a typical gate)
  Leakage cost: 5-10x for that cell
  Area: unchanged (same cell footprint)

Option B: Upsize bottleneck cell from X1 to X2
  Delay improvement: ~30-40% (~15-20ps)
  Leakage cost: ~2x (double the transistor width)
  Area: ~2x for that cell

Option C: Both (LVT + upsize)
  Delay improvement: ~50-60%
  Leakage cost: 10-20x
  Area: 2x

For minimal leakage impact: prefer upsizing (linear leakage increase)
For minimal area impact: prefer Vt swap (no area change)
For maximum speed: both
```

---

## 8. Body Biasing

### 8.1 Forward Body Bias (FBB)

Apply a small positive voltage to the body (P-well for NMOS):

```
Vbs > 0 for NMOS (body voltage higher than source)

Effect: Vth decreases (by body effect coefficient):
  Vth = Vth0 - gamma * (sqrt(2*phi_f + Vsb) - sqrt(2*phi_f))

With FBB (Vsb negative, i.e., Vbs positive):
  Vth decreases -> faster switching, higher leakage
```

Typical FBB: +100 to +300 mV. Delay improvement: 5-15%. Leakage increase: 2-5x.

### 8.2 Reverse Body Bias (RBB)

Apply a negative voltage to the body (for NMOS):

```
Vbs < 0 for NMOS

Effect: Vth increases -> slower switching, lower leakage
```

Typical RBB: -100 to -500 mV. Delay degradation: 10-30%. Leakage reduction: 5-20x.

### 8.3 Adaptive Body Biasing (ABB)

Combine with DVFS: use FBB in active mode (speed up) and RBB in sleep mode (reduce leakage).

```
Active:  VDD = 0.8V, FBB = +200mV -> maximum performance
Idle:    VDD = 0.6V, RBB = -300mV -> minimum leakage
```

Note: Body biasing is less effective in FinFET (the body is fully depleted, no back-gate coupling). It's primarily used in planar CMOS (65nm-28nm). FinFET-based approaches use different Vt implants instead.

---

## 9. Practical Power Budgeting and Thermal Management

### 9.1 Power Budget Example: Mobile SoC

```
Total power budget: 5W (limited by package thermal resistance)

| Block          | % Budget | Power (mW) | Technique Applied      |
|----------------|----------|-----------|------------------------|
| CPU cluster    | 35%      | 1750      | DVFS, power gating     |
| GPU            | 30%      | 1500      | DVFS, power gating     |
| Memory ctrl    | 10%      | 500       | Clock gating           |
| Video codec    | 8%       | 400       | Power gating when idle |
| Display ctrl   | 5%       | 250       | Clock gating           |
| Audio DSP      | 2%       | 100       | Power gating           |
| Modem          | 7%       | 350       | DVFS                   |
| Always-on      | 3%       | 150       | Ultra-low Vt, low freq |

Dynamic vs Leakage split at 7nm:
  Dynamic: ~55% = 2750 mW
  Leakage: ~45% = 2250 mW
```

### 9.2 Thermal Runaway

```
Leakage increases with temperature.
Leakage power generates heat.
Heat increases temperature.
Temperature increases leakage.
...positive feedback loop!

If the thermal design cannot remove heat fast enough:
  T_junction keeps rising -> leakage keeps increasing -> thermal runaway -> chip damage

Thermal equilibrium condition:
  P_dissipated * R_thermal = T_junction - T_ambient
  P_total(T) = P_dynamic + P_leakage(T)

  At equilibrium: P_total(T_j) * R_th = T_j - T_amb
  This equation may have NO solution if leakage growth rate exceeds
  heat removal rate -> thermal runaway

Prevention:
  - Dynamic Thermal Management (DTM): monitor T_junction with on-chip sensors
  - If T > threshold: reduce frequency, reduce voltage, throttle workload
  - Hardware protection: if T > critical, force shutdown
  - Adequate package thermal design: R_thermal low enough
```

### 9.3 Dynamic Thermal Management (DTM) Flow

```
On-chip temp sensors (distributed across die, 1 per ~1mm^2)
         |
         v
  Temperature monitor HW -> interrupt to firmware/OS
         |
         v
  DTM algorithm:
    if T > 85C:  Reduce to OPP_LOW (light throttle)
    if T > 95C:  Reduce to OPP_ULTRA_LOW (heavy throttle)
    if T > 105C: Shut down non-essential domains
    if T > 115C: Emergency shutdown (hardware trip)
```

---

## 10. Power Analysis Methodology

### 10.1 Vector-Based Power Analysis

Uses actual switching activity from simulation:

```tcl
# Run simulation with representative workload
# Generate SAIF (Switching Activity Interchange Format) or VCD

# In PrimeTime PX or Cadence Voltus:
read_verilog design.v
read_sdc design.sdc
read_parasitics design.spef
read_activity design.saif -format saif

report_power -hierarchy
```

**Accuracy:** +/- 10-15% of silicon (depends on workload representativeness).

### 10.2 Vectorless Power Analysis

Uses statistical estimation without simulation:

```tcl
# Set default switching activity
set_switching_activity -static_probability 0.5 -toggle_rate 0.1 [all_nets]

# Override for specific nets with known activity
set_switching_activity -toggle_rate 0.5 [get_nets clk_tree*]
set_switching_activity -toggle_rate 0.02 [get_nets config_reg*]

report_power
```

**Accuracy:** +/- 30-50% (use for early estimation only).

### 10.3 IR Drop Analysis

```
Static IR drop: Average current * mesh resistance
  Target: < 5% of VDD (e.g., < 40mV at 0.8V)

Dynamic IR drop: Transient current surge * (mesh R + L)
  Occurs when many gates switch simultaneously (e.g., clock edge)
  Can cause 2-3x more voltage drop than static
  Target: < 10% of VDD

Tools: Cadence Voltus, Synopsys PrimePower, ANSYS RedHawk
```

---

## 11. Low-Power Design Flow

```
Architecture Definition
  |-- Power budgeting (per block)
  |-- Choose power management strategy (DVFS, PG, CG)
  v
RTL Design
  |-- Clock gating (manual + auto-insert)
  |-- Operand isolation
  |-- Memory partitioning
  |-- Retention register identification
  v
UPF Specification
  |-- Power domains, supplies, states
  |-- Isolation, retention, level shifters
  v
Power-Aware Simulation
  |-- Simulate power-on/off sequences
  |-- Verify isolation, retention, level shifting
  v
Synthesis
  |-- Multi-Vt assignment
  |-- Clock gating insertion
  |-- Cell sizing optimization
  v
Floorplanning
  |-- Power domain placement
  |-- Power grid planning (per domain)
  |-- Switch cell placement
  v
Place & Route
  |-- Isolation cell insertion and placement
  |-- Level shifter insertion
  |-- Retention cell swapping
  |-- Power switch (header/footer) insertion
  |-- Power grid routing
  v
Power Analysis & IR Drop
  |-- Vector-based dynamic power
  |-- Leakage power at all corners
  |-- Static and dynamic IR drop
  |-- EM (electromigration) analysis
  v
Signoff
  |-- All power checks clean
  |-- IR drop within budget
  |-- EM clean
  |-- Thermal analysis OK
```

---

## 12. Interview Questions & Answers

### Q1: Derive P = alpha * C * V^2 * f from first principles.

**A:** Consider a node with load capacitance C. During a 0->1 transition, charge Q = CV flows from VDD through the PMOS to charge C. Energy from VDD = Q * VDD = CV^2. Of this, (1/2)CV^2 is stored in C, and (1/2)CV^2 is dissipated in the PMOS channel. During 1->0, the stored (1/2)CV^2 is dissipated in the NMOS. Total energy per full cycle = CV^2, all dissipated as heat. With switching activity alpha (probability of 0->1 per clock) and clock frequency f, power = alpha * C * V^2 * f. This is exact regardless of transistor sizes (energy loss is intrinsic to charging a capacitor through a resistor).

### Q2: Why does leakage increase exponentially with temperature? Quantify it.

**A:** Sub-threshold leakage is I_sub = I_0 * exp(-Vth / (n * kT/q)). Temperature appears in two places: (1) the thermal voltage Vt = kT/q increases linearly with T, making the exponent less negative; (2) Vth decreases with T at about -1 to -2 mV/K, further reducing the exponent magnitude. Both effects increase leakage exponentially. Rule of thumb: leakage roughly doubles per 10C rise. From 25C to 125C (100C delta), leakage increases by 2^10 ~ 1000x. In practice, the increase is somewhat less (~100-300x) due to mobility reduction at high temperatures partially compensating, but it remains a dominant concern.

### Q3: Explain the difference between clock gating and power gating. When do you use each?

**A:** Clock gating stops the clock to idle blocks, eliminating switching power (dynamic) but leaving supply connected (leakage unchanged). It's lightweight: single ICG cell, ~1 cycle enable latency, no isolation or retention needed. Use for blocks that idle frequently for short periods (tens of cycles). Power gating disconnects VDD entirely, eliminating BOTH dynamic and leakage power. It requires header/footer switches, isolation cells, retention registers, and a power-on sequence (microseconds of latency). Use for blocks that idle for long periods (milliseconds+) where leakage savings justify the overhead. In a mobile SoC, both are used: clock gating for fine-grain (register-level) and power gating for coarse-grain (block-level).

### Q4: Size a header switch for a 2mm^2 domain with 200mA peak current.

**A:** Target IR drop budget: 5% of VDD = 5% * 0.8V = 40mV. Required switch resistance: R = V/I = 40mV / 200mA = 0.2 ohm. For a PMOS header in 7nm with R_per_um ~ 200 ohm*um: total width = 200/0.2 = 1000 um. With standard header cells of 10um width each: need 100 header cells. Distribute them uniformly along the VDD rail across the 2mm^2 domain. For rush current: C_domain ~ 20nF (10nF/mm^2), with 10us ramp time: I_rush = 20nF * 0.8V / 10us = 1.6mA (easily handled by 100 header cells). Use daisy-chain turn-on with 1us delay between groups of 10 cells.

### Q5: Explain sub-threshold slope and the Boltzmann tyranny. Why does it matter for low-power design?

**A:** Sub-threshold slope (SS) is the gate voltage change needed to reduce drain current by 10x: SS = n*kT/q*ln(10) = 60mV/decade minimum at room temperature (when n=1). This means to maintain a 10^6 Ion/Ioff ratio, we need Vth > 6 * 60mV = 360mV. This sets a floor on VDD (need VDD > Vth + overdrive for adequate speed). Since P ~ V^2, this floor limits how low we can scale power. At 7nm, SS is typically 65-70 mV/decade. Novel devices like tunnel FETs (TFETs) and negative capacitance FETs (NCFETs) aim to break this 60mV/decade limit to enable lower VDD operation.

### Q6: What is GIDL and when does it matter?

**A:** GIDL (Gate-Induced Drain Leakage) occurs when the gate is at 0V and drain is at VDD, creating a strong electric field at the gate-drain overlap region. This field causes band-to-band tunneling -- electrons tunnel from the valence band to the conduction band, generating electron-hole pairs. GIDL matters in: (1) DRAM, where it discharges the storage capacitor (reduces retention time); (2) FinFET devices with thin gate oxide and wrap-around gate (larger overlap area); (3) power-gated domains where header switches have high Vds when the domain is off. GIDL is typically 5-10% of total leakage but can be significant at low temperatures (where sub-threshold leakage drops but GIDL stays roughly constant).

### Q7: Describe the complete power-up sequence for a power-gated domain. Why does ordering matter?

**A:** The sequence is: (1) Deassert SLEEP (enable header switches, daisy-chain for rush current control); (2) Wait for virtual VDD to stabilize (monitor ramp or use fixed delay); (3) Assert RESET to all FFs in the domain; (4) Deassert RESET synchronously (to avoid recovery/removal violations); (5) Deassert ISOLATION (outputs now driven by live logic); (6) Assert RESTORE (retention FFs recover saved state); (7) Deassert RESTORE; (8) Resume operation. Ordering is critical: if ISOLATION is deasserted before power is stable, the domain outputs are floating/garbage and corrupt the always-on domain. If RESTORE happens before RESET, the reset overwrites the restored values. If RESET is deasserted asynchronously, FFs may violate recovery/removal timing.

### Q8: What is UPF and how does it differ from RTL?

**A:** UPF (Unified Power Format, IEEE 1801) is a separate TCL-based specification that defines power intent: power domains, supply networks, power states, isolation, retention, level shifters, and power switches. RTL describes function; UPF describes how power is managed. RTL cannot express concepts like "this block can be powered off" or "these outputs need isolation when the domain is off." UPF is read by synthesis tools (to insert special cells), simulation tools (to model power-down behavior), and P&R tools (to physically implement the power architecture). Without UPF, the tools would not know which cells need always-on supply, where to place header switches, or which outputs to isolate.

### Q9: Compare leakage reduction techniques: multi-Vt, power gating, and body biasing.

**A:** Multi-Vt: assigns HVT to non-critical paths (5-10x leakage reduction per cell, no area overhead, works always, standard flow). Power gating: turns off entire domains (nearly 100% leakage elimination when off, but requires isolation/retention/switches, only works for long idle periods). Body biasing: applies reverse body bias to increase Vth in idle mode (5-20x reduction, requires bias generators and distribution network, less effective in FinFET). Best practice: use multi-Vt always (baseline), add power gating for large blocks with long idle periods, use body biasing for fine-grain control in planar CMOS. In modern FinFET designs, multi-Vt + power gating is the dominant combination.

### Q10: How do you estimate the break-even time for power gating?

**A:** Power gating has overhead: energy to power-down (save state, ramp switch), energy to power-up (ramp, restore, re-initialize), and always-on leakage of retention FFs and isolation cells. Break-even time = overhead energy / leakage power saved. Example: if power-up/down costs 10nJ and leakage saved is 5mW, break-even = 10nJ / 5mW = 2us. If the block idles for less than 2us, power gating wastes energy. In practice, add margin: only gate if expected idle > 3-5x break-even. Software/firmware must predict idle duration, which is why DVFS (lower latency) is preferred for short idle periods and power gating for long idle periods.

### Q11: Explain IR drop and its impact on timing. How is it analyzed?

**A:** IR drop is voltage reduction across the power distribution network (PDN) due to resistive losses. If VDD at a cell drops from 0.80V to 0.76V (50mV drop), the cell's delay increases because drive current is proportional to (VDD-Vth)^alpha. A 50mV drop at 0.8V could increase delay by 5-10%. Static IR drop uses average current; dynamic IR drop captures transient surges (e.g., at clock edges when thousands of FFs switch simultaneously). Dynamic IR drop can be 2-3x worse than static. Analysis tools (Voltus, RedHawk) take switching activity + PDN model and compute per-instance voltage, which is fed back to STA for timing derating. Worst-case dynamic IR drop determines if the PDN needs more straps, wider metals, or additional decoupling capacitance.

### Q12: What is the relationship between activity factor and different types of signals?

**A:** Clock signals: alpha = 1.0 (toggle every cycle by definition). Data signals: alpha = 0.05-0.3 (depends on workload, randomness). Address buses: alpha ~ 0.2-0.5 (depends on access pattern). Reset signals: alpha ~ 0 (toggle only at reset). For power estimation, clock network alpha=1.0 makes it the dominant dynamic power consumer (30-50% of total). This is why clock gating is the single most effective dynamic power reduction technique. Accurate alpha values require simulation with representative workloads; using default alpha=0.5 can overestimate power by 2-3x.

### Q13: How do high-k dielectrics reduce gate leakage? Explain with the EOT concept.

**A:** Gate leakage (tunneling) depends exponentially on oxide thickness: thinner oxide = exponentially more tunneling. But thinner oxide gives higher gate capacitance = better transistor control. High-k dielectrics (like HfO2, k~25 vs SiO2 k~3.9) provide the same capacitance with a physically thicker layer: C = epsilon*k/T. EOT (Equivalent Oxide Thickness) = T_physical * (3.9/k_highk). Example: 5nm HfO2 has EOT = 5*(3.9/25) = 0.78nm. This gives the capacitance of a 0.78nm SiO2 gate but the tunneling barrier of a 5nm film, reducing gate leakage by 100-1000x. High-k was introduced at the 45nm node (Intel 2007) and enabled continued gate length scaling.

### Q14: What is thermal runaway and how do you prevent it in a design?

**A:** Thermal runaway is a positive feedback loop: leakage generates heat, heat increases temperature, higher temperature increases leakage exponentially. If the cooling system cannot remove heat fast enough, temperature rises until the chip fails. Prevention: (1) Design the PDN and package for adequate thermal dissipation (R_thermal * P_max < T_max - T_ambient); (2) Use on-chip temperature sensors (one per major block) feeding a Dynamic Thermal Management (DTM) controller; (3) DTM reduces frequency/voltage when temperature exceeds thresholds; (4) Hardware thermal trip circuit forces emergency shutdown above critical temperature (typically 125C junction); (5) At design time, perform thermal simulations to verify the power budget at worst-case workload doesn't exceed cooling capacity.

### Q15: Explain operand isolation with a concrete example showing power savings.

**A:** Consider a 32-bit multiplier used only when `valid` is high (25% of cycles). Without operand isolation, random input toggling causes the multiplier's internal nodes to switch uselessly 75% of the time. With operand isolation, the inputs are AND-gated with `valid`:

```verilog
wire [31:0] a_gated = valid ? a : 32'd0;
wire [31:0] b_gated = valid ? b : 32'd0;
wire [63:0] product = a_gated * b_gated;
```

When valid=0, inputs are 0, and the multiplier internals don't toggle (0*0 = 0, no transitions). Power savings: the multiplier has ~10,000 internal nodes with average alpha=0.2. Without isolation: P_mult = 0.2 * C * V^2 * f * 100% of time. With isolation: P_mult = 0.2 * C * V^2 * f * 25% + P_mux_overhead. Savings ~ 75% of multiplier dynamic power minus the small MUX (AND gate) overhead. The 64 AND gates add negligible power compared to the 10,000-node multiplier.

### Q16: How does memory partitioning save power?

**A:** A 64KB SRAM as one monolithic block draws full power on every access (all bitlines charge, all sense amps fire). Partitioned into 8 banks of 8KB: only the accessed bank is active (1/8 the bitline + sense amp power). Additional savings: shorter bitlines = lower capacitance per access, smaller decoders, banks in standby can be put in retention or shut down. Trade-off: bank selection logic adds a small amount of area and delay, and the total area increases slightly due to duplicated peripheral circuits. But for memories > 16KB, partitioning almost always wins. Advanced memory compilers offer configurable banking ratios.

### Q17: What are the challenges of near-threshold computing?

**A:** Operating at VDD near Vth (e.g., 0.4V with Vth = 0.35V): (1) Ion/Ioff ratio is very poor (~10-100x) making logic unreliable; (2) Process variation causes huge delay spread (Vth variation of +/-30mV at 3-sigma means some gates are super-threshold and others are sub-threshold simultaneously); (3) SRAM fails first (6T SRAM read/write margins collapse near Vth, need 8T or 10T cells); (4) Frequency drops to MHz range; (5) Leakage becomes comparable to dynamic power, so the energy advantage plateaus. Used in ultra-low-power applications (IoT sensors, biomedical) where throughput is not critical. Design requires wide timing margins, oversized transistors, and error-resilient architectures.

### Q18: Explain the difference between static and dynamic IR drop. Which is worse?

**A:** Static IR drop: caused by average DC current flowing through the resistive power grid. It's the steady-state voltage drop. Calculated as V_drop = I_avg * R_mesh. Typically 10-30mV for a well-designed grid. Dynamic IR drop: caused by sudden current surges (e.g., when the clock edge triggers thousands of FFs switching simultaneously). The transient current causes both resistive drop (I*R) and inductive voltage spikes (L*dI/dt). Dynamic IR drop is typically 2-5x worse than static and can cause 50-100mV droops lasting a few hundred picoseconds. These droops slow down gates exactly when they need to switch, causing timing violations that static analysis misses. This is why dynamic IR drop simulation with realistic switching activity is mandatory for signoff.

### Q19: How does leakage scale across technology nodes?

**A:** Leakage per transistor increased dramatically from 180nm to 65nm due to: thinner gate oxide (more tunneling), shorter channels (worse DIBL), lower Vth (needed to maintain speed as VDD scaled). At 45nm, high-k/metal gate reduced gate leakage significantly. At 22nm, FinFET reduced sub-threshold leakage (better electrostatic control, n closer to 1.0, steeper SS). However, total chip leakage continued to grow because transistor count increased faster than per-transistor leakage decreased. At 7nm, total leakage is ~45-50% of total power for a high-performance design, making leakage management (multi-Vt, power gating) mandatory, not optional.

### Q20: What power verification checks must pass before tapeout?

**A:** (1) UPF consistency: all domains defined, all crossings have level shifters and/or synchronizers; (2) Isolation completeness: every output of every switchable domain is isolated; (3) Retention coverage: all required state registers have retention; (4) Power-aware simulation: correct behavior during all power-on/off transitions; (5) Power grid EM clean: no metal wire exceeds electromigration current density limit; (6) Static IR drop < 5% VDD; (7) Dynamic IR drop < 10% VDD; (8) Rush current within package current limit; (9) Total power within thermal budget at worst-case workload; (10) Leakage power within budget at worst-case temperature; (11) No missing supply connections; (12) All power switches correctly sized and connected.
