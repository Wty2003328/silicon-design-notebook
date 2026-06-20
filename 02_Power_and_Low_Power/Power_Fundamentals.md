# Power Fundamentals in Digital IC / ASIC Design

## 1. Why Power Matters

Power consumption is the dominant constraint in modern ASIC and SoC design. It determines
battery life (mobile), package cost (thermal dissipation capability), cooling requirements
(heatsink, fan, liquid cooling), reliability (electromigration, hot-carrier injection), and
ultimately the end-user experience. As technology scales below 28nm, power has supplanted
area and even timing as the primary limiter -- a phenomenon known as the "power wall."

Key metrics every engineer must internalize:

| Metric               | Definition                                         | Why It Matters                                    |
|----------------------|----------------------------------------------------|---------------------------------------------------|
| **Average power**    | Time-averaged power over a workload                | Battery life, thermal steady-state, TDP rating    |
| **Peak power**       | Maximum instantaneous power draw                   | IR drop, supply integrity, package current limit   |
| **Power density**    | Power per unit area (W/mm^2)                       | Hotspot formation, reliability, thermal runaway   |
| **Energy per op**    | Energy consumed per useful operation (pJ/op)       | True efficiency metric for accelerators           |
| **Leakage power**    | Power consumed when circuit is idle (no switching) | Standby battery drain, always-on domain budgets   |

---

## 2. Total Power Equation

```
P_total = P_dynamic + P_static
P_total = P_switching + P_short_circuit + P_leakage
P_total = (alpha * C_L * V_DD^2 * f_clk) + (I_SC * V_DD) + (I_leak * V_DD)
```

| Component       | Typical Share (65nm) | Typical Share (7nm) | Trend                        |
|-----------------|----------------------|---------------------|------------------------------|
| Switching       | 60-70%               | 40-50%              | Decreasing fraction          |
| Short-circuit   | 5-10%                | 5-10%               | Roughly stable               |
| Leakage         | 20-30%               | 40-50%              | Increasing (until FinFET)    |

---

## 3. Switching Power -- Derivation from First Principles

### 3.1 The RC Charging/Discharging Circuit

Consider a CMOS inverter driving a load capacitance C_L. During a 0-to-1 output transition,
the PMOS transistor turns on and charges C_L through its channel resistance R_p:

```
           Vdd
            |
         [R_p]    (PMOS on-resistance)
            |
            +------ Vout
            |
          [C_L]
            |
           GND
```

The voltage across C_L rises exponentially:

```
V_out(t) = Vdd * (1 - e^(-t/(R_p * C_L)))
```

Energy delivered by the supply during charging:

```
E_supply = integral(0 to inf) { Vdd * i(t) dt }
         = integral(0 to inf) { Vdd * C_L * dV/dt * dt }
         = Vdd * C_L * integral(0 to Vdd) { dV }
         = C_L * Vdd^2
```

Energy stored on the capacitor after charging:

```
E_cap = (1/2) * C_L * Vdd^2
```

Energy dissipated in the PMOS resistance during charging:

```
E_PMOS = E_supply - E_cap = C_L * Vdd^2 - (1/2)*C_L*Vdd^2 = (1/2)*C_L*Vdd^2
```

During a 1-to-0 transition, the NMOS turns on and discharges C_L. The stored energy
(1/2)*C_L*Vdd^2 is dissipated in the NMOS channel resistance. So for one complete
cycle (0->1->0):

```
E_cycle = E_PMOS_charging + E_NMOS_discharging
        = (1/2)*C_L*Vdd^2 + (1/2)*C_L*Vdd^2
        = C_L * Vdd^2
```

### 3.2 Why It Is C*V^2 (Not 1/2 * C * V^2)

This is a classic interview question. The answer depends on what you mean by "per transition"
versus "per cycle":

- **Energy per full cycle** (0->1->0) = C_L * Vdd^2
- **Energy per transition** (one edge) = (1/2) * C_L * Vdd^2

The factor of 2 appears or disappears depending on whether you count one transition or a
full switching cycle. In the standard power equation:

```
P_switching = alpha * C_L * Vdd^2 * f_clk
```

Here, **alpha** is the activity factor defined as the average number of 0->1 transitions
per clock cycle (NOT total transitions). Since only the 0->1 transition draws energy from
Vdd (the 1->0 transition dissipates stored energy to ground), we get:

```
P = alpha * (1/2 * C_L * Vdd^2 * 2) * f = alpha * C_L * Vdd^2 * f
```

Some textbooks write P = (1/2) * alpha * C_L * Vdd^2 * f where alpha counts BOTH 0->1 and
1->0 transitions per cycle. Both are equivalent -- the factor of 1/2 either sits in the
energy term or the activity term. Be clear about your definition in interviews.

### 3.3 Activity Factor (alpha) in Detail

For a signal with static probability p (probability of being 1):

```
alpha = 2 * p * (1-p) * f_clk    [if alpha counts both transitions]
alpha = p * (1-p) * f_clk        [if alpha counts only 0->1]
```

For random data (p=0.5): alpha = 0.25 (one transition out of every 4 clock cycles on average
if counting 0->1 only) or alpha = 0.5 (both transitions).

Important: Clock signals have alpha = 1.0 (toggle every half-cycle on both edges), which is
why clock networks consume 30-50% of total dynamic power.

### 3.4 Numerical Example

```
Given:
  28nm design, 10 million equivalent gates
  Average switching activity alpha = 0.15 (realistic for data path)
  Average load capacitance per gate = 1.2 fF
  Vdd = 0.9V
  Clock frequency = 500 MHz

P_switching = alpha * C_total * Vdd^2 * f
  C_total = 10M * 1.2fF = 12 nF
  P_switching = 0.15 * 12e-9 * (0.9)^2 * 500e6
              = 0.15 * 12e-9 * 0.81 * 500e6
              = 0.15 * 4.86e-3
              = 729 uW ... wait, let me recalculate

  P_switching = 0.15 * 12e-9 * 0.81 * 500e6
              = 0.15 * 12 * 0.81 * 500 * (1e-9 * 1e6)
              = 0.15 * 12 * 0.81 * 500 * 1e-3
              = 0.15 * 4860 * 1e-3
              = 0.15 * 4.86
              = 0.729 W

So roughly 730 mW of switching power.
```

Add clock network (~40% of dynamic power) and this block might draw ~1.2W dynamic total.

---

## 4. Short-Circuit Power

### 4.1 The Physical Mechanism

During an input transition on a CMOS gate, there is a brief period when BOTH the PMOS and
NMOS transistors are simultaneously conducting. This creates a direct path from Vdd to GND
(a "short circuit") and draws crowbar current.

```
        Vdd
         |
      [PMOS]---+
         |     |
  Vin ---+     +--- Vout
         |     |
      [NMOS]---+
         |
        GND

  When Vin is between Vtn and (Vdd - |Vtp|):
    - NMOS is ON (Vgs_n > Vtn)
    - PMOS is ON (|Vgs_p| > |Vtp|)
    - Both conduct simultaneously
```

### 4.2 Voltage Transfer Characteristic (VTC) and the Overlap Window

```
  Vdd|          ___________
     |         /
     |        /    <-- Both ON in this transition region
     |       /         (Vtn < Vin < Vdd-|Vtp|)
 Vout|      /
     |     /
     |____/
     +-----|------|-------> Vin
          Vtn  Vdd-|Vtp|
```

The short-circuit current flows during the time the input voltage is between Vtn and
(Vdd - |Vtp|). For Vdd = 0.8V and Vth = 0.3V:

```
Short-circuit window = Vdd - 2*Vth = 0.8 - 2*0.3 = 0.2V
```

### 4.3 Analytical Model for Short-Circuit Power

For a symmetric inverter with matched PMOS/NMOS strengths and a linear input ramp:

```
I_peak = (beta / 12) * (Vdd - 2*Vth)^3

where:
  beta = mu * Cox * (W/L)  (transconductance parameter)
  
P_sc = (beta / 12) * (Vdd - 2*Vth)^3 * tau * f

where:
  tau = input transition time (rise or fall time)
  f = switching frequency
```

### 4.4 Numerical Example

```
Given:
  Vdd = 0.8V, Vth = 0.3V (both NMOS and PMOS)
  beta = 200 uA/V^2 (typical for minimum-size in 28nm)
  tau_rise = tau_fall = 50 ps
  f = 1 GHz
  C_L = 2 fF

P_sc = (200e-6/12) * (0.8 - 0.6)^3 * 50e-12 * 1e9
     = 16.67e-6 * 8e-3 * 50e-3
     = 16.67e-6 * 4e-4
     = 6.67 nW per gate

P_switching = 0.5 * 2e-15 * (0.8)^2 * 1e9 = 0.5 * 2e-15 * 0.64 * 1e9 = 640 nW

P_sc / P_switching = 6.67 / 640 = ~1%
```

For this example, short-circuit power is about 1% of switching power. In practice, with
slow input transitions (large tau) or when Vdd >> 2*Vth, short-circuit can be 5-15%.

### 4.5 Key Design Implications

- **Matched rise/fall times minimize short-circuit power.** If input rise time >> fall time
  (or vice versa), the slow edge causes prolonged overlap.
- **When Vdd < 2*Vth, short-circuit power is zero.** Both devices cannot be on simultaneously.
  This happens in ultra-low-voltage (near-threshold) designs.
- **Fast transitions help.** Strong drivers with sharp edges reduce the overlap window.
- **Short-circuit power is proportional to tau (transition time)**, while switching power is
  independent of tau. As loads get heavier (slower transitions), short-circuit fraction grows.

---

## 5. Sub-threshold Leakage -- MOSFET Physics

### 5.1 The Sub-threshold Current Equation

Below threshold (Vgs < Vth), the MOSFET is in weak inversion. Current does not abruptly
go to zero -- it decays exponentially:

```
I_ds = I_0 * exp((Vgs - Vth) / (n * V_T)) * (1 - exp(-Vds / V_T))

where:
  I_0   = technology-dependent reference current
        = mu * Cox * (W/L) * (n-1) * V_T^2
  n     = subthreshold swing ideality factor (1.3 - 1.5 for bulk, ~1.1 for FinFET)
  V_T   = thermal voltage = kT/q
        = 26 mV at T = 300K (27C)
        = 33.5 mV at T = 125C (398K)
  Vth   = threshold voltage (depends on process, body bias, temperature)
  Vgs   = gate-to-source voltage
  Vds   = drain-to-source voltage
```

### 5.2 Understanding Each Parameter

**Subthreshold slope (S):**
```
S = n * V_T * ln(10) = n * 2.3 * V_T

At 300K: S = 1.3 * 2.3 * 26mV = ~78 mV/decade (bulk CMOS)
For FinFET: S = 1.1 * 2.3 * 26mV = ~66 mV/decade (closer to ideal 60 mV/decade)
```
This means: reducing Vgs by 78mV reduces leakage by 10x in bulk CMOS.

**Ideality factor (n):**
```
n = 1 + C_depletion / C_oxide

For bulk CMOS: C_dep/Cox is significant -> n = 1.3-1.5
For FinFET: gate wraps channel, C_dep/Cox is small -> n ~ 1.05-1.1
For GAAFET: even better electrostatic control -> n ~ 1.02-1.05
```

### 5.3 Temperature Dependence

Leakage has a strong exponential temperature dependence through two mechanisms:

1. **V_T = kT/q increases with temperature** -- wider thermal distribution of carriers
2. **Vth decreases with temperature** -- approximately -1 to -2 mV/C

Combined effect:

```
I_leak(T2) / I_leak(T1) = exp((Vth(T1) - Vth(T2)) / (n * V_T(T2)))
                          * (V_T(T2) / V_T(T1))^2

Rule of thumb: leakage approximately DOUBLES for every 10-12C increase in
junction temperature for typical processes.

Example:
  At 25C:  I_leak = 10 nA/um
  At 85C:  I_leak = 10 * 2^((85-25)/10) = 10 * 2^6 = 640 nA/um
  At 125C: I_leak = 10 * 2^((125-25)/10) = 10 * 2^10 = ~10 uA/um
```

This is why leakage power analysis must be done at the worst-case temperature corner
(typically 125C for commercial, 105C for industrial).

### 5.4 Drain-Induced Barrier Lowering (DIBL)

DIBL is a short-channel effect where the drain voltage influences the source-channel barrier:

```
Vth_effective = Vth0 - eta * Vds

where:
  eta = DIBL coefficient (typically 20-100 mV/V for planar, 10-30 mV/V for FinFET)
```

Physical explanation:
- In a short-channel device, the drain depletion region extends closer to the source
- This lowers the potential barrier at the source, allowing more carriers to flow
- Higher Vds -> lower effective Vth -> more subthreshold leakage

```
Example:
  Vth0 = 0.35V, eta = 50 mV/V
  At Vds = 0.1V: Vth_eff = 0.35 - 0.005 = 0.345V
  At Vds = 0.8V: Vth_eff = 0.35 - 0.040 = 0.310V

  Leakage ratio = exp((0.345 - 0.310) / (1.3 * 0.026))
                = exp(0.035 / 0.0338)
                = exp(1.035)
                = 2.8x increase due to DIBL alone
```

### 5.5 Body Effect

The threshold voltage depends on the source-to-body voltage (Vsb):

```
Vth = Vth0 + gamma * (sqrt(2*phi_f + Vsb) - sqrt(2*phi_f))

where:
  gamma = body effect coefficient = sqrt(2*q*epsilon_si*N_A) / Cox
        (typically 0.3 - 0.5 V^(1/2) for bulk CMOS)
  phi_f = Fermi potential = (V_T) * ln(N_A / n_i)
        (typically 0.3 - 0.4 V)
```

Implications for power:
- **Stacked NMOS** (series transistors): bottom transistor has Vsb > 0 due to virtual
  ground node, increasing Vth and reducing leakage. This is the "stack effect" -- a
  2-input NAND leaks less than a 2-input NOR in an NMOS pull-down stack.
- **Reverse body bias (RBB):** applying positive Vsb to NMOS increases Vth, reduces leakage.
  Used as active leakage reduction technique.
- **Forward body bias (FBB):** applying negative Vsb to NMOS decreases Vth, increases speed
  but also increases leakage. Used for performance boosting.

### 5.6 Leakage Numbers Across Technology Nodes

| Node   | Vdd   | I_leak (NMOS, SVT, per um width) | Total chip leakage (typical SoC) | Notes                        |
|--------|-------|----------------------------------|----------------------------------|------------------------------|
| 130nm  | 1.2V  | ~1 nA/um                         | < 100 mW                        | Leakage not yet dominant     |
| 65nm   | 1.0V  | ~10 nA/um                        | 200-500 mW                      | Leakage becoming significant |
| 45nm   | 0.9V  | ~50-100 nA/um                    | 500 mW - 1W                     | Multi-Vt essential           |
| 28nm   | 0.9V  | ~100-200 nA/um                   | 1-3W                            | Power gating widespread      |
| 16nm FF| 0.8V  | ~5-10 nA/um                      | 0.5-2W                          | FinFET dramatically helps    |
| 7nm FF | 0.75V | ~10-20 nA/um                     | 1-3W                            | More transistors offset gain |
| 5nm FF | 0.7V  | ~15-30 nA/um                     | 2-5W                            | Density drives total leakage |
| 3nm GAA| 0.65V | ~10-20 nA/um                     | 2-5W                            | GAAFET helps electrostatics  |

Note: per-transistor leakage decreased with FinFET (16nm) but total chip leakage kept growing
because transistor count scaled faster.

---

## 6. Gate Oxide Tunneling Leakage

### 6.1 Quantum Mechanical Tunneling

As gate oxide thickness scaled below ~3nm (starting at 90nm node), electrons can quantum-
mechanically tunnel through the thin SiO2 barrier. This is a fundamentally different mechanism
from subthreshold leakage.

### 6.2 Direct Tunneling vs Fowler-Nordheim Tunneling

```
Direct Tunneling (thin oxide, < ~3nm):
  - Electron tunnels directly through the full oxide barrier
  - Current is exponential in oxide thickness: I ~ exp(-alpha * t_ox)
  - Dominant mechanism for modern thin oxides
  - Cannot be reduced by lowering Vdd (barrier shape doesn't change much)

Fowler-Nordheim Tunneling (thick oxide, > ~3nm, high electric field):
  - High gate voltage creates a triangular barrier
  - Electron tunnels through the triangular portion
  - Dominant in flash memory programming (intentionally used)
  - Not significant in normal digital CMOS operation at modern Vdd
```

### 6.3 Gate Leakage Numbers

```
Pure SiO2:
  t_ox = 5.0 nm: I_gate ~ 10^(-5) A/cm^2   (negligible)
  t_ox = 2.0 nm: I_gate ~ 1 A/cm^2          (problematic)
  t_ox = 1.2 nm: I_gate ~ 100 A/cm^2        (unacceptable)
```

### 6.4 High-k Dielectrics

The solution was to replace SiO2 (k ~ 3.9) with high-k materials:

```
HfO2 (k ~ 22): can use physically thicker oxide while maintaining same
capacitance (same "electrical thickness" or EOT)

EOT = t_high-k * (k_SiO2 / k_high-k)

Example:
  Physical HfO2 thickness = 4 nm
  EOT = 4 * (3.9 / 22) = 0.71 nm equivalent SiO2

  4nm physical barrier -> dramatically reduced tunneling current
  0.71nm equivalent oxide thickness -> same capacitance as ultra-thin SiO2
```

Intel introduced HfO2/metal gate at 45nm (2007). This reduced gate leakage by
~100x compared to SiO2 of equivalent EOT.

---

## 7. Gate-Induced Drain Leakage (GIDL)

### 7.1 Physical Mechanism

GIDL occurs at the gate-drain overlap region when:
- Gate voltage is LOW (0V)
- Drain voltage is HIGH (Vdd)

This creates a high electric field in the gate-drain overlap that causes band-to-band
tunneling (BTBT):

```
Energy Band Diagram at Gate-Drain Overlap:

                     Gate = 0V        Drain = Vdd
                        |                |
  Ec _______________    |    ___         |     ___________
                    \   |   /   \        |    /
                     \__|__/     \       |   /
                        |        \______|__/
  Ev _______________    |               |     ___________
                        |               |
                     oxide          overlap
                                    region

  The valence band on the channel side aligns with the conduction band
  on the drain side -> band-to-band tunneling of electrons
```

### 7.2 When GIDL Matters

- **SRAM cells:** wordline (gate) is LOW during hold, bitline/internal node (drain) is at
  Vdd. GIDL can corrupt stored data.
- **Power-gated domains:** when header/footer switch is off, internal node voltages float --
  GIDL can create unexpected current paths.
- **DRAM retention:** GIDL at the access transistor can discharge the storage capacitor.

### 7.3 GIDL Mitigation

- Lightly doped drain (LDD) extensions reduce the overlap field
- Underlap devices (gate does not fully overlap drain) reduce GIDL at cost of performance
- FinFET naturally reduces GIDL due to better gate control and thinner body

---

## 8. Junction Leakage

### 8.1 Reverse-Biased PN Junction

Every MOSFET has PN junctions at source/drain to substrate (or well). When reverse-biased,
these junctions conduct a small reverse current:

```
I_junction = I_s * (exp(V_forward / V_T) - 1)

For reverse bias (V < 0):
I_junction ~ -I_s = -A * J_s

where:
  A = junction area
  J_s = saturation current density
      ~ 10^(-7) A/cm^2 at 25C for modern processes (much less than subthreshold)
```

### 8.2 Temperature Dependence

Junction leakage has even stronger temperature dependence than subthreshold leakage:

```
I_junction(T) ~ T^2 * exp(-E_g / (2*k*T))

where E_g = bandgap energy of silicon (1.12 eV)

Roughly doubles every 8-10C (faster than subthreshold).
```

At room temperature, junction leakage is typically 10-100x smaller than subthreshold leakage.
But at high temperatures (125C+), it can become significant, especially for large diffusion
areas.

---

## 9. Glitch Power (Hazard-Related Switching)

### 9.1 What Causes Glitches

Glitches (spurious transitions) occur due to unbalanced path delays through combinational logic,
particularly at reconvergent fanout points:

```
               +------[delay=2]------+
   Input A --->|                     |---> AND ---> Y
               +------[delay=5]------+

   A changes at t=0
   Fast path: input arrives at AND at t=2 -> Y changes
   Slow path: input arrives at AND at t=5 -> Y changes AGAIN (back to original)
   
   Result: Y glitches between t=2 and t=5 even though final value is the same
```

### 9.2 Impact on Power

```
In an unoptimized datapath:
  - Glitch power can be 15-30% of total switching power
  - Multiplier trees are notorious for glitches (many reconvergent paths)
  - Each glitch propagates downstream, causing more glitches (glitch amplification)
```

### 9.3 Timing Diagram

```
        0   2   4   5   7   9
        |   |   |   |   |   |
Input A ____/```````````````````
Fast    ______/```````````````````
Slow    ____________/```````````````
Output  ______/`````\___/```````````
              ^glitch^
              
  This glitch consumes C*Vdd^2 of extra energy for no useful computation.
```

### 9.4 Mitigation Techniques

1. **Path balancing:** equalize delay paths through logic (add buffers on fast paths)
2. **Retiming:** move registers to break long combinational paths
3. **Guard registers:** insert pipeline registers at reconvergent points
4. **Hazard-free logic synthesis:** decompose functions to avoid static hazards
5. **Clock gating downstream logic:** if the glitch happens before the clock edge and
   settles before capture, it doesn't matter for functionality -- but it STILL wastes power

---

## 9B. Technology Scaling Trends -- Dennard Scaling and Its Breakdown

### 9B.1 Dennard Scaling (1974-2005)

Robert Dennard's 1974 paper established that as MOSFET dimensions shrink by a factor k:
- W, L -> W/k, L/k (gate length and width shrink)
- Vdd -> Vdd/k (voltage scales proportionally)
- Cox -> k * Cox (thinner oxide, higher capacitance per area)
- I_ds -> I_ds/k (current scales with voltage and dimensions)
- Delay -> Delay/k (circuits get faster)
- Power density remains CONSTANT (more transistors, same power per area)

The math: Power per gate = C * V^2 * f. After scaling: (C/k) * (V/k)^2 * (k*f) = C*V^2*f / k.
But k times as many gates fit in the same area: total power density = (C*V^2*f/k) * k = C*V^2*f = constant.

This was the foundation of "frequency doubles every generation" for 30 years.

### 9B.2 Why Dennard Scaling Broke (~65nm)

Dennard scaling requires Vdd to scale with Vth. But as Vth drops, leakage increases exponentially:

```
For Vth = 0.3V at 130nm:
  If we scale Vdd from 1.2V to 0.6V (k=2), Vth should also halve to 0.15V.
  But at Vth = 0.15V: I_off = I_0 * exp(-0.15 / (1.3 * 0.026)) = I_0 * exp(-4.44)
  vs Vth = 0.3V:       I_off = I_0 * exp(-0.30 / (1.3 * 0.026)) = I_0 * exp(-8.88)
  
  Ratio = exp(4.44) = 85x more leakage per transistor!

  With 2x more transistors per area: total leakage density increases by ~170x.
  The chip would be entirely leakage-dominated -- unusable.

Resolution: Vth stopped scaling below ~0.25-0.35V.
Consequence: Vdd could not scale as Dennard predicted (Vdd/Vth ratio dropped).
  130nm: Vdd = 1.2V, Vth = 0.35V, overdrive = 0.85V
   65nm: Vdd = 1.0V, Vth = 0.30V, overdrive = 0.70V  (Vdd didn't halve!)
   28nm: Vdd = 0.9V, Vth = 0.28V, overdrive = 0.62V
    7nm: Vdd = 0.75V, Vth = 0.25V, overdrive = 0.50V
```

The fundamental limit is the **Boltzmann tyranny** (60 mV/decade subthreshold swing): you cannot
reduce Vth below ~0.25V without I_off exceeding design limits.

### 9B.3 Consequences of Broken Dennard Scaling

```
1. Power density INCREASES with scaling (instead of staying constant):
   Each generation: ~2x more transistors, Vdd drops only ~10-15% (not 30%)
   Power density grows ~30-50% per generation

2. Dark silicon: A chip cannot power all its transistors simultaneously.
   At 7nm, only ~50-70% of a chip can be active at full frequency.
   The rest must be power-gated or clock-gated ("dark silicon").
   
   Example: Apple A15 (5nm, ~15B transistors):
     Full activation would draw ~30-50W (exceeds mobile thermal budget of ~5-8W)
     Only the active cores and accelerators are powered at any time

3. Frequency plateaued: Clock frequency stalled at ~3-5 GHz around 2005
   - Further frequency increase requires higher Vdd -> exponentially more power
   - Instead: more cores, wider SIMD, specialized accelerators
   - The industry moved to multi-core (parallelism) instead of frequency scaling

4. Voltage scaling slowed dramatically:
   250nm → 130nm: Vdd dropped from 2.5V → 1.2V (halved in 2 generations)
   130nm → 7nm:   Vdd dropped from 1.2V → 0.75V (only 38% in 5+ generations)
```

### 9B.4 FinFET Advantage for Leakage

```
Why FinFET (16nm and below) partially restored scaling:

Planar MOSFET at 20nm:
  Subthreshold slope = 80-100 mV/decade (n = 1.3-1.5)
  DIBL = 50-100 mV/V
  Vth variability = +/- 30-50 mV (random dopant fluctuation)

FinFET at 16nm:
  Subthreshold slope = 65-70 mV/decade (n = 1.05-1.15)
  DIBL = 20-30 mV/V
  Vth variability = +/- 15-25 mV (undoped channel)
  
Consequence: FinFET can use LOWER Vth at same leakage budget:
  Planar at Vth = 0.35V: I_off = I_0 * exp(-0.35/(1.35*0.026)) = I_0 * exp(-9.97)
  FinFET at Vth = 0.28V: I_off = I_0 * exp(-0.28/(1.10*0.026)) = I_0 * exp(-9.79)
  
  Similar I_off, but FinFET's lower Vth gives (0.75-0.28)/(0.75-0.35) = 1.175x more
  overdrive at Vdd = 0.75V. This means faster at same leakage, or same speed at lower Vdd.
  
  FinFET enabled Vdd to drop from ~0.9V (28nm planar) to ~0.75V (7nm FinFET)
  while maintaining performance. Without FinFET, the 7nm node would not be viable.
```

### 9B.5 Dark Silicon and Power Density Limits

```
Power density ceiling for different cooling:
  Passive (mobile, no fan):     ~5 W/cm^2  -> ~5-8W total
  Active air cooling (laptop):  ~30 W/cm^2 -> ~45-65W total
  Active air cooling (desktop): ~80 W/cm^2 -> ~150-250W total
  Liquid cooling:               ~200 W/cm^2 -> ~300-500W total

At 7nm with ~100M gates/mm^2:
  If ALL gates switch at 2 GHz with alpha = 0.1:
  P = 0.1 * 100e6 * 1.2e-15 * 0.75^2 * 2e9 = 13.5 W/mm^2 = 1350 W/cm^2
  
  This is 10x beyond even liquid cooling!
  Hence: only a fraction of the chip can be active at any time.

Dark silicon fraction (fraction that must be off at peak performance):
  45nm planar: ~30% dark
  16nm FinFET: ~40% dark
   7nm FinFET: ~50% dark
   3nm GAA:    ~55-60% dark
  
  This drives architecture: heterogeneous cores, power-gated accelerators,
  near-threshold computing for background tasks.
```

### 9B.6 Worked Leakage Calculation With All Parameters

```
Calculate subthreshold leakage for a single NMOS in 28nm at 85C:

Given:
  W/L = 2.0 um / 0.028 um (minimum-size inverter NMOS)
  Vth0 = 0.32V (SVT at 25C)
  n = 1.25 (bulk CMOS)
  mu = 300 cm^2/V*s (electron mobility at 85C)
  Cox = 25 fF/um^2 (equivalent)
  Vds = 0.9V (Vdd)
  Vgs = 0V (transistor is OFF)
  eta_DIBL = 60 mV/V
  gamma = 0.4 V^(1/2)
  Vsb = 0V (body tied to source)

Step 1: Temperature effects on Vth and Vt
  V_T(85C = 358K) = k*358/q = 1.381e-23 * 358 / 1.602e-19 = 30.8 mV
  Vth(85C) = Vth0 - 2mV/K * (85-25) = 0.32 - 0.12 = 0.20V

Step 2: DIBL effect
  Vth_eff = Vth(85C) - eta * Vds = 0.20 - 0.060 * 0.9 = 0.20 - 0.054 = 0.146V

Step 3: I_0 calculation
  I_0 = mu * Cox * (W/L) * (n-1) * V_T^2
      = 300e-4 * 25e-15/1e-12 * (2.0e-6/0.028e-6) * 0.25 * (30.8e-3)^2
      = 300e-4 * 25e-3 * 71.4 * 0.25 * 9.49e-4
      = 300e-4 * 25e-3 * 71.4 * 2.37e-4
      = 300e-4 * 4.24e-4
      = 1.27e-5 A/um ... let me simplify

  Simplified: I_0 ~ 0.5 uA per um of width for 28nm SVT
  For W = 2.0 um: I_0 = 1.0 uA

Step 4: Subthreshold leakage
  I_sub = I_0 * exp((Vgs - Vth_eff) / (n * V_T)) * (1 - exp(-Vds / V_T))
        = 1.0e-6 * exp((0 - 0.146) / (1.25 * 0.0308)) * (1 - exp(-0.9/0.0308))
        = 1.0e-6 * exp(-0.146 / 0.0385) * (1 - ~0)     [Vds >> V_T, so exp(-Vds/Vt) ~ 0]
        = 1.0e-6 * exp(-3.79)
        = 1.0e-6 * 0.0225
        = 22.5 nA per transistor

Step 5: Power per transistor
  P_leak = I_sub * Vdd = 22.5e-9 * 0.9 = 20.3 nW per transistor

Step 6: Scale to a 10M-gate block
  Total leakage power = 10e6 * 20.3 nW = 203 mW at 85C

  Compare with 25C (Vth_eff ~ 0.32 - 0.054 = 0.266V, V_T = 25.9mV):
  I_sub_25C = 1.0e-6 * exp(-0.266 / (1.25*0.0259)) = exp(-8.22) = 0.27e-3
  = 0.27 nA per transistor
  P_leak_25C = 10e6 * 0.27e-9 * 0.9 = 2.4 mW at 25C

  Ratio: 203/2.4 = 85x from 25C to 85C (consistent with ~2x per 10C rule: 2^6 = 64,
  actual ratio is higher due to the Vth temperature coefficient being more aggressive here)

This calculation shows why leakage analysis MUST be done at worst-case temperature.
A design that meets power budget at 25C will fail catastrophically at 85C.
```

---

## 10. Power at Advanced Technology Nodes

### 10.1 FinFET Advantages (16nm/14nm/7nm/5nm)

FinFET dramatically improved the power-performance trade-off:

```
Planar MOSFET:       Gate controls channel from ONE side
                     -> poor electrostatics at short gate lengths
                     -> high subthreshold slope (~80-100 mV/decade)
                     -> high DIBL (~50-100 mV/V)

FinFET:              Gate wraps channel on THREE sides
                     -> excellent electrostatic control
                     -> subthreshold slope ~65-70 mV/decade
                     -> DIBL ~20-30 mV/V
                     -> Can use lower Vth for same leakage -> faster or lower Vdd
```

Power impact compared to equivalent planar node:
- ~50% reduction in dynamic power at same performance
- ~80-90% reduction in leakage at same Vth target
- Enables lower Vdd operation (0.7-0.8V vs 0.9-1.0V)

### 10.2 Gate-All-Around (GAAFET) / Nanosheet (3nm and below)

```
GAAFET:              Gate wraps channel on ALL FOUR sides
                     -> Best possible electrostatic control
                     -> Subthreshold slope ~62-65 mV/decade (near ideal)
                     -> DIBL ~10-15 mV/V
                     -> Variable channel width via nanosheet width adjustment
                        (unlike FinFET where width is quantized to fin pitch)
```

### 10.3 Backside Power Delivery Network (BSPDN)

Traditional approach: power and signal routing share the same metal stack (front side).
Power rails consume routing resources and have high IR drop.

Backside power delivery (Intel PowerVia, TSMC backside):
- Power rails routed on the backside of the wafer (through TSVs)
- Signal routing on the front side gets more resources
- IR drop reduced by ~30-50% (shorter, wider power paths)
- Enables denser standard cell libraries
- Allows thinner front-side metal stack (lower parasitic capacitance -> less switching power)

### 10.4 Complementary FET (CFET) and Stacked Transistors

Future (2nm and beyond): stack NMOS directly on top of PMOS:
- ~50% area reduction per logic cell
- Shorter interconnects -> lower capacitance -> lower dynamic power
- Potential for ~30% power reduction at same performance vs GAAFET
- Manufacturing complexity is extreme (precise alignment, thermal budget)

---

## 11. Power Measurement and Activity Annotation

### 11.1 SAIF Format (Switching Activity Interchange Format)

SAIF captures the switching activity of every net in the design. It records toggle count,
time spent at logic-0 (T0), and time spent at logic-1 (T1) over a simulation period.

```
(SAIF
  (SAIFILE
    (DIRECTION "backward")
    (DESIGN "my_chip_top")
    (DATE "2026-01-15")
    (VENDOR "Synopsys")
    (PROGRAM_NAME "VCS")
    (VERSION "2024.06")
    (DIVIDER /)
    (TIMESCALE 1 ps)
    (DURATION 1000000)
    (INSTANCE my_chip_top
      (NET
        (clk
          (T0 500000) (T1 500000) (TC 1000) (IG 0)
        )
        (reset_n
          (T0 100000) (T1 900000) (TC 2) (IG 0)
        )
        (data_bus\[0\]
          (T0 480000) (T1 520000) (TC 250) (IG 0)
        )
        (data_bus\[1\]
          (T0 510000) (T1 490000) (TC 312) (IG 0)
        )
      )
      (INSTANCE cpu_core
        (NET
          (alu_out\[0\]
            (T0 420000) (T1 580000) (TC 1450) (IG 2)
          )
        )
      )
    )
  )
)
```

Field explanations:
- **T0:** Total time (in timescale units) the net was at logic 0
- **T1:** Total time the net was at logic 1
- **TC:** Toggle Count -- number of transitions (both 0->1 and 1->0)
- **IG:** Ignore count -- number of unknown/X state transitions
- **DURATION:** Total simulation time

From SAIF, tools derive:
```
Static probability (SP) = T1 / DURATION
Toggle rate (TR) = TC / DURATION     (transitions per time unit)
Switching activity = TC / (2 * num_clock_cycles)  (probability of transition per clock)
```

### 11.2 VCD vs FSDB

| Feature          | VCD (Value Change Dump)      | FSDB (Fast Signal Database)         |
|------------------|------------------------------|-------------------------------------|
| Standard         | IEEE 1364 (open)             | Proprietary (Synopsys/Verdi)        |
| File size        | Very large (text format)     | Compressed (5-10x smaller)          |
| Typical size     | 10-100 GB for large designs  | 1-20 GB for same design             |
| Signal access    | Sequential only              | Random access (can jump to time)    |
| 4-value support  | Yes (0, 1, X, Z)            | Yes (plus strengths, analog)        |
| Use in power     | Gate-level time-based        | Preferred for large designs          |
| Generation       | $dumpvars in testbench       | $fsdbDumpvars (Synopsys PLI)        |

### 11.3 Activity Annotation Flow

```
  RTL Simulation (VCS/Xcelium)
      |
      |  Run representative workload (boot + application scenario)
      |  Dump switching activity for the analysis window
      v
  Activity File (SAIF / VCD / FSDB)
      |
      v
  Power Analysis Tool (PrimeTime PX / Voltus)
      |
      |  Read gate-level netlist
      |  Read parasitic data (SPEF)
      |  Annotate activity from simulation onto netlist
      |  Calculate power per cell, per net, per hierarchy
      v
  Power Reports (per-instance, per-module, per-power-domain)
```

### 11.4 Vectorless Power Estimation

When simulation is not available (or too expensive), tools estimate activity:

**How it works:**
1. Assign default toggle rate to primary inputs (typically TR=0.1 to 0.5)
2. Propagate activity through the netlist using probabilistic analysis
3. At each gate: SP_out = f(SP_inputs), TR_out = f(TR_inputs, gate_type)

**When vectorless is accurate:**
- For clock tree power (activity is known = 1.0 per clock cycle)
- When using aggressive toggle rates for worst-case estimation
- For relative comparisons between design alternatives

**When vectorless is misleading:**
- For low-activity modules (reset logic, configuration registers)
- For data-dependent activity (multipliers, encoders)
- For modules with enable signals (vectorless may not model the enable correctly)

**Hybrid approach (recommended for real projects):**
- Annotate SAIF on critical blocks (CPU core, memory controllers, data paths)
- Use vectorless for low-power blocks and glue logic
- Compare vectorless vs annotated results for a few blocks to calibrate

---

## 12. Power Analysis Tool Flow

### 12.1 PrimeTime PX (Synopsys) -- Average Power

```tcl
# ========================================
# PrimeTime PX Average Power Analysis Flow
# ========================================

# 1. Read design
set search_path ". /libs/28nm/db"
set link_library "* 28nm_svt_tt_0p9v_25c.db"
read_verilog my_design_netlist.v
current_design my_chip_top
link_design

# 2. Read timing constraints (needed for clock definition)
read_sdc my_design.sdc

# 3. Read parasitics (post-layout)
read_parasitics my_design.spef.gz

# 4. Set operating conditions
set_operating_conditions tt_0p9v_25c

# 5. Read switching activity (choose one)
# Option A: SAIF annotation
read_saif my_simulation.saif -strip_path testbench/dut

# Option B: VCD annotation
# read_vcd my_simulation.vcd -strip_path testbench/dut

# Option C: Vectorless (set default activity)
# set_switching_activity -static_probability 0.5 -toggle_rate 0.1 -base_clock clk [all_inputs]

# 6. Perform power analysis
update_power

# 7. Generate reports
report_power -hierarchy > power_hierarchy.rpt
report_power -cell_power > power_cells.rpt
report_power -net_power > power_nets.rpt
report_power -verbose > power_detailed.rpt

# 8. Check for unannotated nets
report_switching_activity -unannotated > unannotated.rpt
```

### 12.2 PrimeTime PX -- Time-Based Power Analysis

```tcl
# ========================================
# Time-Based Power Analysis (power waveform)
# ========================================

# Read design (same as above)
# ...

# Read VCD (SAIF is NOT sufficient -- need time-resolved data)
read_vcd my_simulation.vcd -strip_path testbench/dut

# Set time-based analysis mode
set power_enable_analysis true
set_power_analysis_options -waveform_interval 1ns

# Run analysis
update_power

# Report time-based power
report_power -time_based > power_waveform.rpt

# Get peak power
report_power -peak > peak_power.rpt
```

### 12.3 Voltus (Cadence) Power Analysis Flow

```tcl
# ========================================
# Cadence Voltus Power Analysis Flow
# ========================================

# Read design database
read_design -physical_data my_design.oa

# Or from DEF/LEF
read_lef /libs/28nm/lef/tech.lef
read_lef /libs/28nm/lef/stdcell.lef
read_def my_design.def
read_verilog my_design.v
set_top_module my_chip_top

# Read parasitics
read_spef my_design.spef

# Read timing (for clock propagation)
read_sdc my_design.sdc

# Set power analysis mode
set_power_analysis_mode -method static  ;# or dynamic_vectorbased
set_power_output_dir ./power_results

# Read activity
read_activity_file -format SAIF my_simulation.saif -scope testbench/dut

# Run power analysis
report_power -outfile power_report.txt

# For dynamic (time-based) analysis:
set_power_analysis_mode -method dynamic_vectorbased
read_activity_file -format VCD my_simulation.vcd -scope testbench/dut
set_dynamic_power_simulation -resolution 1ns
report_power -outfile power_dynamic.txt
```

### 12.4 Average vs Time-Based vs Peak Analysis

| Analysis Type | Input Activity | Output                      | Use Case                            |
|---------------|----------------|-----------------------------|-------------------------------------|
| Average       | SAIF           | Single power number (mW)    | Power budgeting, battery life       |
| Time-based    | VCD/FSDB       | Power vs time waveform      | IR drop analysis, peak detection    |
| Peak          | VCD/FSDB       | Maximum instantaneous power | Supply design, worst-case IR drop   |

---

## 13. Interview Questions and Answers

### Q1: "A 28nm design has 10M gates, average switching activity 0.15, average gate capacitance 1.2fF, Vdd=0.9V, 500MHz. Estimate dynamic power."

**A:** P = alpha * C * Vdd^2 * f = 0.15 * (10e6 * 1.2e-15) * 0.81 * 500e6 = 0.15 * 12e-9 * 0.81 * 5e8 = 729 mW. Add ~40% for clock network: ~1.02W total dynamic power. This is the data-path + clock estimate; real chips would also include memory macro dynamic power.

### Q2: "Why does leakage double every ~10C? Derive it."

**A:** I_leak ~ exp(-Vth/(n*kT/q)). As T increases: (1) kT/q increases, directly increasing the exponential argument denominator (but the numerator also decreases because Vth drops with T). For a 10C increase from T to T+10: Vth drops by ~15-20mV, and V_T = kT/q increases by ~0.87mV. The net effect is approximately: I(T+10)/I(T) = exp(delta_Vth/(n*V_T)) ~ exp(15e-3 / (1.3*26e-3)) ~ exp(0.44) ~ 1.6-2.0x. The exact ratio depends on the Vth temperature coefficient and the node. Typical industry rule: 2x per 10C.

### Q3: "Explain why the energy per switching event is C*Vdd^2, not 1/2*C*Vdd^2."

**A:** During 0->1 transition, Vdd delivers C*Vdd^2 energy. Capacitor stores (1/2)*C*Vdd^2. The other half is dissipated in PMOS. During 1->0, the stored (1/2)*C*Vdd^2 is dissipated in NMOS. Per complete cycle: C*Vdd^2 from supply, all dissipated as heat. The (1/2) factor appears only if you count a single transition. The activity factor definition determines which convention you use.

### Q4: "What is DIBL and how does it affect power?"

**A:** DIBL = Drain-Induced Barrier Lowering. In short-channel devices, Vds reduces the source-channel barrier, lowering effective Vth. Higher Vds -> lower Vth -> exponentially more subthreshold leakage. DIBL coefficient eta is ~50-100 mV/V in planar, ~20-30 mV/V in FinFET. This means at Vds=0.8V, effective Vth can be 40-80mV lower than at Vds=0V in planar devices, causing 3-10x more leakage. FinFET's superior electrostatics reduce DIBL significantly.

### Q5: "A chip has 2W leakage at 85C. Estimate leakage at 25C."

**A:** Using the 2x per 10C rule: temperature difference = 60C, so factor = 2^(60/10) = 2^6 = 64. Leakage at 25C = 2W / 64 = 31.25 mW. Note: this is approximate -- the actual ratio depends on process specifics, but 64x is a reasonable estimate.

### Q6: "Why did the industry move to high-k dielectrics?"

**A:** SiO2 gate oxide had to be thinned to ~1.2nm at the 90nm node to maintain gate capacitance (and thus drive current) as channel lengths shrank. At this thickness, direct quantum mechanical tunneling caused gate leakage current of ~100 A/cm^2, making gate leakage comparable to subthreshold leakage. HfO2 (k~22 vs 3.9 for SiO2) allows a physically thicker oxide (e.g., 4nm HfO2 ~ 0.7nm SiO2 equivalent) with the same capacitance, reducing tunneling current by ~100x. Intel introduced high-k/metal gate at 45nm (2007).

### Q7: "What is the stack effect and how does it help leakage?"

**A:** In series-connected transistors (e.g., NAND pull-down), when one transistor is off, the intermediate node floats to a voltage between 0 and Vdd. This causes the off transistor to have: (1) positive Vsb -> higher Vth via body effect, and (2) reduced Vds -> reduced DIBL. Both effects reduce leakage. A 2-stack reduces leakage by ~10x compared to a single transistor. 3-stack gives ~25-50x reduction. This is why NAND gates are preferred over NOR gates for low-leakage design (NMOS stack in NAND benefits from stack effect more than PMOS stack in NOR because PMOS has higher Vth already).

### Q8: "What fraction of switching power is due to glitches in a typical design?"

**A:** 15-30% in unoptimized datapaths (e.g., ripple-carry adders, multiplier trees). After optimization (path balancing, retiming), this can be reduced to 5-10%. Clock gating downstream logic eliminates the power impact of glitches on registered outputs but not on the combinational logic itself. Synthesis tools automatically minimize glitches through technology mapping, but physical design (routing delays) can introduce new glitch paths.

### Q9: "Compare leakage mechanisms: subthreshold vs gate vs junction vs GIDL. Which dominates when?"

**A:** At room temperature, 7nm: subthreshold >> gate (mitigated by high-k) > GIDL > junction. At 125C: subthreshold >>> junction (grows fastest with T) > GIDL > gate (temperature-insensitive). For HVT cells at room temp: subthreshold is suppressed, GIDL can become comparable. For ultra-low-voltage near-threshold designs: subthreshold dominates overwhelmingly (exponentially sensitive to Vgs near Vth).

### Q10: "Explain forward vs backward SAIF annotation."

**A:** Forward annotation: simulate RTL, dump SAIF at RTL level, then map net names from RTL to gates during power analysis. This requires name mapping (can be lossy). Backward annotation: tools read the gate-level netlist, generate a "SAIF template" with gate-level net names, you simulate the gate-level netlist (or annotated RTL) and fill in the template. Backward is more accurate because net names match exactly. Forward is faster because RTL simulation is much faster than gate-level simulation.

### Q11: "How would you estimate the power of a block you haven't designed yet?"

**A:** (1) Use a similar block from a previous project as a reference and scale by gate count, activity, Vdd, and frequency. (2) Use high-level estimation: P = alpha * N_gates * C_avg * Vdd^2 * f, where N_gates and C_avg come from synthesis estimates or published data. (3) For memory-dominated blocks, estimate memory macro power from datasheets and add logic power. (4) Industry rules of thumb: ARM Cortex-A class cores are ~100-300 mW/GHz at 7nm; DSP blocks are ~50-100 mW/GHz; always-on domains target < 1 mW.

### Q12: "What is the difference between average power and peak power? Why do both matter?"

**A:** Average power (mW over a workload period) determines thermal steady state and battery life. Peak power (instantaneous maximum, often over a 1-10ns window) determines: (1) worst-case IR drop on the power grid (V_drop = I_peak * R_grid), (2) Ldi/dt noise on the supply, (3) maximum current the package must deliver. Peak can be 3-5x average. Both must meet specs: average for thermal/battery, peak for supply integrity.

### Q13: "If you reduce Vdd by 10%, what happens to dynamic power, leakage, and frequency?"

**A:** Dynamic power: P ~ Vdd^2, so 0.9^2 = 0.81 -> 19% reduction. Leakage: subthreshold leakage has weak Vdd dependence (mainly through DIBL: lower Vds -> slightly higher Vth -> less leakage, maybe 10-20% reduction, highly process-dependent). Gate leakage reduces roughly proportionally with Vdd. Frequency: Fmax ~ (Vdd-Vth)^alpha / Vdd where alpha~1.3. For Vdd=0.9V->0.81V, Vth=0.3V: (0.81-0.3)^1.3/0.81 vs (0.9-0.3)^1.3/0.9 -> roughly 15-20% frequency reduction. Net: 19% power savings for ~15-20% performance loss -- this is why DVFS is so effective.

### Q14: "Why is clock power so high?"

**A:** Clock toggles every cycle (activity factor = 1.0, vs 0.05-0.2 for data). Clock networks are heavily buffered (thousands of buffers in clock tree) with large drive strengths. Clock wires run long distances across the chip with significant wire capacitance. In a typical SoC: clock tree = 30-50% of dynamic power. This is why clock gating is the single most impactful power reduction technique.

### Q15: "What is the minimum energy point (MEP) and why does it exist?"

**A:** As you reduce Vdd, dynamic power drops quadratically, but the circuit slows down, so you need more time (cycles) to do the same work. At very low Vdd (near Vth), the circuit is so slow that leakage energy per operation dominates (leakage power * long execution time). The MEP is the Vdd where total energy per operation (dynamic + leakage) is minimized. It typically occurs at Vdd ~ Vth (near-threshold computing). Below MEP, reducing Vdd actually increases total energy because the performance loss causes leakage to dominate. MEP is ~0.3-0.4V for most processes.

### Q16: "Explain the power-performance-area (PPA) trade-off at the cell level."

**A:** A larger cell (wider transistors) has: more drive current -> faster (better P), but more capacitance -> more switching power (worse P), and more area (worse A). Multi-Vt: LVT is faster (better performance) but leaks ~5-20x more than HVT (worse leakage power), at the same area. Sizing: upsizing a cell on the critical path improves timing but increases its capacitance and the capacitance it presents to its driver. The optimization is a constrained problem: minimize power subject to timing constraints, using cell sizing and Vt assignment as the primary levers.

---

## 14. Summary: Power Breakdown Decision Framework

```
When analyzing power for a new design, think:

1. DYNAMIC POWER (~50-60% at modern nodes)
   |-- Switching: alpha * C * Vdd^2 * f
   |   |-- Clock tree: 30-50% of dynamic (alpha=1)
   |   |-- Data path: proportional to switching activity
   |   |-- Memory: access power dominates for SRAM/register files
   |   \-- Glitch power: 5-15% in optimized designs
   |
   \-- Short-circuit: ~5-10% of dynamic (minimize with fast transitions)

2. STATIC/LEAKAGE POWER (~40-50% at 7nm and below)
   |-- Subthreshold: dominant component, exponential in Vth and temperature
   |-- Gate tunneling: mitigated by high-k but still present
   |-- GIDL: important for SRAM and power-gated designs
   \-- Junction: significant only at high temperatures

3. KEY KNOBS FOR REDUCTION
   |-- Vdd reduction: quadratic dynamic, moderate leakage
   |-- Clock gating: targets the largest dynamic component
   |-- Multi-Vt: directly trades leakage for performance
   |-- Power gating: eliminates leakage in unused blocks
   \-- Activity reduction: operand isolation, data gating
```

---

## 15. Numbers to Memorize

| Quantity | Value | Why it matters |
|---|---|---|
| Dynamic : static split (≤7nm) | ~50–60% : 40–50% | leakage is now a co-equal term |
| Switching share, 65nm → 7nm | 60–70% → 40–50% of total | shrinking fraction as nodes scale |
| Leakage share, 65nm → 7nm | 20–30% → 40–50% | rises until FinFET reins it in |
| Clock-tree share of dynamic | 30–50% | α=1 on every clock node → gate it first |
| Glitch power | 5–15% of dynamic | path balancing reduces it |
| Short-circuit power | ~5–10% of dynamic (≈1% with fast edges) | minimize with sharp transitions |
| Switching energy/cycle | **α·C·Vdd²·f** (full C·V², not ½) | charge *and* discharge each cycle |
| Subthreshold swing | ~78 mV/dec bulk, ~66 FinFET (ideal 60) | one decade of leakage per S |
| Thermal voltage kT/q | 26 mV @ 27 °C, 33.5 mV @ 125 °C | sets the swing |
| Leakage vs temperature | ~**2× per 10–12 °C** | the thermal-runaway risk |
| Vth temperature coefficient | −1 to −2 mV/°C | Vth falls → leakage climbs with heat |
| Vdd as a knob | **quadratic** on dynamic, moderate on leakage | the single biggest lever |

---

*This document targets senior-engineer / staff-level ASIC power interview preparation.
All numerical examples use realistic process parameters. Cross-reference with
Power_Reduction_Techniques.md, UPF_Power_Intent.md, and Power_Analysis_and_Signoff.md.*
