# CMOS Fundamentals and Device Physics — Senior Engineer Deep Dive

## Table of Contents
1. MOSFET Operation
2. CMOS Inverter and VTC
3. Noise Margins
4. Propagation Delay and Power-Delay Product
5. CMOS Logic Families
6. Latch-Up
7. ESD Protection
8. FinFET and Advanced Nodes
9. Process Variations and DTCO
10. Interview Q&A (25+ Questions)

---

## 1. MOSFET Operation

### 1.1 NMOS Transistor — Regions of Operation

```
                  Gate (G)
                   |
            ───────┤ oxide (SiO2, tox)
           |       |       |
    Source (S)    body    Drain (D)
      n+     p-substrate    n+
```

**Three operating regions (NMOS, VGS > 0, VDS > 0):**

```
Cutoff (VGS < Vth):
  IDS ≈ 0   (no channel formed)
  Transistor is OFF (acts as open switch)

Linear/Triode (VGS > Vth, VDS < VGS - Vth):
  IDS = μn * Cox * (W/L) * [(VGS - Vth) * VDS - VDS²/2]
  Transistor acts like a voltage-controlled resistor
  Ron ≈ 1 / [μn * Cox * (W/L) * (VGS - Vth)]

Saturation (VGS > Vth, VDS ≥ VGS - Vth):
  IDS = (1/2) * μn * Cox * (W/L) * (VGS - Vth)² * (1 + λ*VDS)
  Channel pinched off at drain end
  Current approximately constant (controlled by VGS)
  λ = channel-length modulation parameter (1/VA)
```

**Key parameters:**
```
μn:    Electron mobility (~400 cm²/V·s for bulk Si at 300K)
Cox:   Gate oxide capacitance = εox / tox
W/L:   Width-to-length ratio (designer's knob)
Vth:   Threshold voltage (~0.3-0.5V for modern processes)
λ:     Channel-length modulation (~0.05-0.1 V⁻¹)
```

### 1.2 PMOS Transistor

Same equations but with opposite signs/polarities:
```
Cutoff:      VGS > Vth (Vth < 0 for PMOS, so |VGS| < |Vth|)
Linear:      |VGS| > |Vth|, |VDS| < |VGS| - |Vth|
Saturation:  |VGS| > |Vth|, |VDS| ≥ |VGS| - |Vth|

Hole mobility μp ≈ 150-200 cm²/V·s (about 2-3x lower than electrons)
→ PMOS must be 2-3x wider than NMOS for equal drive strength
```

### 1.3 Threshold Voltage

```
Vth = Vth0 + γ * (√(2*φF + VSB) - √(2*φF))

Vth0:  Zero-bias threshold voltage
γ:     Body-effect coefficient = √(2*q*εsi*NA) / Cox
φF:    Fermi potential = (kT/q) * ln(NA/ni)
VSB:   Source-body voltage (body effect)
```

**Body effect:** When VSB > 0 (source above body potential), Vth increases.
This is important in stacked transistors (NAND/NOR gates) where the transistor
closest to VDD/GND has VSB = 0 but upper transistors have VSB > 0.

### 1.4 Short-Channel Effects (Modern Nodes)

```
DIBL (Drain-Induced Barrier Lowering):
  Drain voltage reduces source-channel barrier → Vth decreases with VDS
  Vth(VDS) = Vth(long) - η * VDS
  η ≈ 50-100 mV/V for 7nm

Velocity Saturation:
  At high lateral field, carrier velocity saturates at vsat ≈ 10^7 cm/s
  IDS = W * Cox * vsat * (VGS - Vth - VDSsat)
  Velocity saturation makes current LINEARLY dependent on (VGS - Vth)
  instead of quadratic — critical for advanced nodes

Hot Carrier Injection (HCI):
  High-energy carriers injected into gate oxide → oxide damage → Vth shift
  Worse at high VDS, affects reliability

Gate Leakage (tunneling):
  Below ~2nm oxide thickness, electrons tunnel through gate oxide
  Solution: High-k dielectrics (HfO2, k ≈ 20-25 vs SiO2 k ≈ 3.9)
  EOT (Equivalent Oxide Thickness) = tphysical * (3.9/k)
```

---

## 2. CMOS Inverter — Voltage Transfer Characteristic (VTC)

### 2.1 Circuit Structure

```
         VDD
          |
       ┌──┤ PMOS (W_p/L_p)
       │  |
  IN ──┤  ├── OUT
       │  |
       └──┤ NMOS (W_n/L_n)
          |
         GND
```

### 2.2 VTC Analysis — Five Regions

```
Vout
 |
VDD ─────────────╮
 |                │\
 |                │ \   Region B: NMOS sat, PMOS linear
 |                │  \
 |        Region A│   \  Region C: both in saturation
 |   NMOS off,    │    │  (transition region, high gain)
 |   PMOS linear  │    │
 |                │   /  Region D: NMOS linear, PMOS sat
 |                │  /
 |                │ /
 |                │/
 0 ───────────────┴─────── Vin
 0            VM        VDD

 VM = switching threshold (Vin = Vout point)
```

**Five operating regions as Vin sweeps 0 → VDD:**

```
Region A (Vin < Vthn):
  NMOS: cutoff     PMOS: linear
  Vout = VDD (PMOS pulls up, NMOS is off)

Region B (Vthn < Vin < VM, NMOS enters saturation):
  NMOS: saturation  PMOS: linear
  Vout starts to drop

Region C (Vin ≈ VM, both in saturation):
  NMOS: saturation  PMOS: saturation
  Maximum gain |dVout/dVin| → transition region
  This is where the inverter "switches"

Region D (VM < Vin < VDD + Vthp):
  NMOS: linear      PMOS: saturation
  Vout continues dropping

Region E (Vin > VDD + Vthp, noting Vthp < 0):
  NMOS: linear      PMOS: cutoff
  Vout = 0 (NMOS pulls down, PMOS is off)
```

### 2.3 Switching Threshold (VM) Derivation

At VM, Vin = Vout = VM, and NMOS current = PMOS current (both in saturation):

```
(1/2) * kn * (VM - Vthn)² = (1/2) * kp * (VDD - VM - |Vthp|)²

where kn = μn*Cox*(Wn/Ln), kp = μp*Cox*(Wp/Lp)

Let r = √(kp/kn):

VM = (Vthn + r*(VDD - |Vthp|)) / (1 + r)
```

**For symmetric VTC (VM = VDD/2):**
```
kp/kn = 1  →  (μp*Wp/Lp) = (μn*Wn/Ln)
Since μp ≈ μn/2.5:  Wp/Lp ≈ 2.5 * (Wn/Ln)
```

**Numerical example (7nm):**
```
VDD = 0.7V, Vthn = 0.3V, |Vthp| = 0.3V
r = 1 (symmetric sizing):
VM = (0.3 + 1*(0.7-0.3)) / (1+1) = 0.7/2 = 0.35V = VDD/2  ✓
```

---

## 3. Noise Margins

### 3.1 Definition

```
VOH: Output high voltage (Vout when Vin = 0) = VDD
VOL: Output low voltage (Vout when Vin = VDD) = 0
VIH: Input high voltage (min Vin recognized as logic 1)
VIL: Input low voltage (max Vin recognized as logic 0)

VIH and VIL are defined as the points where |dVout/dVin| = 1
(unity gain points on the VTC)

NMH (Noise Margin High) = VOH - VIH
NML (Noise Margin Low)  = VIL - VOL
```

```
Vout
 |
VOH ────╮
 |       \
 |        \←── slope = -1 (VIL point)
 |         \
 |          \←── transition region
 |           \
 |            \←── slope = -1 (VIH point)
 |             ╰────
VOL                    Vin
        VIL  VM  VIH
    |←NML→|      |←NMH→|
    VOL   VIL    VIH   VOH
```

### 3.2 Noise Margin Derivation for Symmetric CMOS Inverter

For a symmetric CMOS inverter (kn = kp, Vthn = |Vthp| = Vt):

```
VIL ≈ (2*Vout + Vin - Vt) / (solving dVout/dVin = -1)

For symmetric inverter (approximation):
  VIL ≈ (3*VDD + 2*Vt) / 8    (for short-channel)
  VIH ≈ (5*VDD - 2*Vt) / 8

  NMH = VDD - VIH = VDD - (5*VDD - 2*Vt)/8 = (3*VDD + 2*Vt)/8
  NML = VIL - 0   = (3*VDD + 2*Vt)/8

  NMH = NML → symmetric noise margins
```

**Numerical example:**
```
VDD = 0.7V, Vt = 0.3V:
  NMH = NML = (3*0.7 + 2*0.3)/8 = (2.1 + 0.6)/8 = 0.3375V

  NM/VDD = 0.3375/0.7 = 48.2% of VDD

  This is high — ideal CMOS has excellent noise margins
  Compare with: NMOS-only logic NM ≈ 20-30% of VDD
```

### 3.3 Factors That Degrade Noise Margins

```
1. Vt mismatch (process variation):
   If Vthn ≠ |Vthp|, VM shifts away from VDD/2
   NMH and NML become unequal

2. VDD scaling:
   As VDD decreases, absolute noise margin decreases
   At VDD = 2*Vt: NM → 0 (circuit doesn't work!)
   This is why Vt must scale with VDD

3. Unequal sizing (kn ≠ kp):
   Shifts VM, one noise margin grows while the other shrinks

4. Supply noise (IR drop):
   Effective VDD decreases → NM decreases
   10% VDD drop → significant NM degradation at advanced nodes

5. Temperature:
   Higher T → lower μ, lower Vth → shifts VTC
```

### 3.4 Regenerative Property

A cascade of CMOS inverters restores logic levels because the gain in the transition
region is much greater than 1:

```
|Gain| = |dVout/dVin| at VM

For long-channel CMOS:
  |Gain| = (kn + kp) * (VDD/2 - Vt) / (λn + λp) * VDD

  Typically |Gain| > 10-50

This means: a signal degraded by noise gets "cleaned up" after
passing through a few inverters → digital logic is noise-tolerant
```

---

## 4. Propagation Delay and Power-Delay Product

### 4.1 Propagation Delay

```
tpHL: Time for output to fall from VOH to VDD/2 (high-to-low)
tpLH: Time for output to rise from VOL to VDD/2 (low-to-high)
tp = (tpHL + tpLH) / 2

tpHL ≈ 0.69 * Rn * CL    (NMOS discharging load capacitance)
tpLH ≈ 0.69 * Rp * CL    (PMOS charging load capacitance)

where:
  Rn = VDD / [kn * (VDD - Vthn)²]  (effective NMOS resistance)
  Rp = VDD / [kp * (VDD - |Vthp|)²]
  CL = Cgate(fanout) + Cwire + Cdrain(self)
```

**For equal rise/fall (tpHL = tpLH):**
```
Rn = Rp → kn(VDD - Vthn)² = kp(VDD - |Vthp|)²
For Vthn = |Vthp|: kn = kp → Wp ≈ 2.5 * Wn (same as symmetric VM)
```

### 4.2 Delay Optimization

```
1. Increase W/L: reduces R → faster, but increases C (area + power)
   Diminishing returns: doubling W doubles both drive and self-load

2. Reduce CL: shorter wires, fewer fanout, buffer insertion
   FO4 delay = delay of inverter driving 4 identical inverters
   FO4 ≈ 15-30 ps at 7nm → fundamental speed metric

3. Increase VDD: reduces R quadratically → faster
   But: power ∝ VDD² → power-performance trade-off

4. Reduce Vt: increases drive current → faster
   But: leakage ∝ exp(-Vt/nVT) → exponential leakage increase
   LVT cells: fast but leaky  |  HVT cells: slow but low-leakage

5. Logical effort: optimize gate sizing for minimum path delay
   Minimum delay when each stage has equal effort
```

### 4.3 Power-Delay Product (PDP) and Energy-Delay Product (EDP)

```
PDP = P_avg × tp
    = (α * CL * VDD² * f) × (k * CL * VDD / ID)
    ∝ CL * VDD²    (energy per transition)

EDP = PDP × tp  ∝ CL² * VDD³ / ID
    → Minimize EDP by finding optimal VDD (below VDD_nominal)

At optimal VDD for minimum EDP:
  VDD_opt ≈ 3 * n * VT ≈ 0.2-0.3V (near-threshold computing)
```

---

## 5. CMOS Logic Families

### 5.1 Static CMOS (Complementary)

```
Structure: Pull-Up Network (PMOS) + Pull-Down Network (NMOS)
           PUN and PDN are complementary (dual networks)

NAND gate (2-input):
         VDD
          |
       ┌──┤ P1 (A)    PMOS in parallel
       ├──┤ P2 (B)
       │  ├── OUT
       └──┤ N1 (A)    NMOS in series
          ├
       ───┤ N2 (B)
          |
         GND

NOR gate (2-input):
         VDD
          |
       ┌──┤ P1 (A)    PMOS in series
          ├
       ├──┤ P2 (B)
       │  ├── OUT
       ├──┤ N1 (A)    NMOS in parallel
       └──┤ N2 (B)
          |
         GND
```

**NAND vs NOR performance:**
```
NAND: NMOS in series → higher PDN resistance → slower pull-down
      PMOS in parallel → lower PUN resistance → faster pull-up
      Net: NAND is preferred over NOR because:
        - NMOS (series) is faster than PMOS (series)
        - For N-input: NMOS series W = N*Wmin, PMOS parallel W = Wmin

NOR:  PMOS in series → much higher PUN resistance (μp is already low)
      For N-input NOR: PMOS series W = N*2.5*Wmin → huge area
      NOR gates are generally avoided for high fan-in
```

### 5.2 Pseudo-NMOS

```
         VDD
          |
       ┌──┤ PMOS (always ON, gate tied to GND)
       │  ├── OUT
       └──┤ PDN (NMOS network, same as static CMOS)
          |
         GND

Advantages: Fewer transistors (N+1 vs 2N), faster for wide NOR/OR
Disadvantages:
  - Static power consumption (when PDN is on, current flows VDD→GND)
  - VOL ≠ 0 (voltage divider between PMOS and PDN)
  - Reduced NML
  - Ratio-ed logic — sizing matters for correct operation
```

### 5.3 Transmission Gate Logic

```
         A ──┤├── B         NMOS
         A ──┤├── B         PMOS (complementary control)
              |
Combined: passes both 0 and 1 without threshold drop

Used in: MUXes, XOR, latches
Advantage: compact, good for pass logic
Disadvantage: no gain → signal degrades through chain of TGs
```

### 5.4 Dynamic Logic (Domino)

```
Phase 1 (CLK = 0, precharge):
  PMOS precharges output node to VDD
  NMOS evaluation network is disconnected (footer off)

Phase 2 (CLK = 1, evaluate):
  NMOS evaluation network conditionally discharges output
  If PDN has path to GND → output goes low
  If no path → output stays high (precharged)

         VDD
          |
       ┌──┤ Precharge PMOS (CLK')
       │  ├── Dynamic node
       └──┤ PDN (NMOS network)
          ├
       ───┤ Footer NMOS (CLK)
          |
         GND

Domino: Add a static inverter after dynamic node
  → output is non-inverting, can cascade
  → but only supports non-inverting logic (AND-OR)
```

**Domino issues:**
```
1. Charge sharing: internal nodes discharge dynamic node → false evaluation
   Fix: precharge internal nodes, add keeper PMOS

2. Clock skew: if evaluate arrives before precharge completes → error
   Fix: careful clock tree design, sufficient precharge time

3. Noise sensitivity: no restoring property during evaluate
   Single event → irreversible discharge
   Fix: keeper (weak PMOS feedback to hold precharged value)

4. Cannot implement inverting functions directly
   Fix: use NP-CMOS (alternating NMOS and PMOS domino stages)
```

---

## 6. Latch-Up

### 6.1 The Parasitic PNPN Structure

In a CMOS inverter, the n-well (for PMOS) and p-substrate (for NMOS) create a parasitic
thyristor (SCR = Silicon Controlled Rectifier):

```
Cross-section:

  VDD (to PMOS source)         GND (to NMOS source)
      |                             |
   ┌──┴──┐                       ┌──┴──┐
   │ p+  │    ┌───────────┐      │ n+  │
   │source│   │  n-well   │      │source│
   └──┬──┘   │           │      └──┬──┘
      │      │  ┌─────┐  │         │
      │      │  │ p+  │  │         │
      │      │  │drain│  │         │
      │      └──┴─────┴──┘         │
      │         p-substrate        │
      │                            │
      └────────────────────────────┘
                  GND (substrate contact)

Parasitic BJTs formed:
  Q1 (PNP): Emitter = PMOS source (p+), Base = n-well, Collector = p-substrate
  Q2 (NPN): Emitter = NMOS source (n+), Base = p-substrate, Collector = n-well

  Rwell: resistance of n-well (base of PNP)
  Rsub:  resistance of p-substrate (base of NPN)
```

```
Equivalent circuit:
                VDD
                 |
             Q1 (PNP)
            E   C
            |   |
            |   ├─── Rsub ──── GND
            |   |
            ├── Rwell
            |   |
            |   C
            E   |
             Q2 (NPN)
                 |
                GND
```

### 6.2 Latch-Up Triggering

```
Normal operation: Both BJTs are OFF (no base current)

Trigger conditions:
1. Current injection into substrate or well
   - ESD event (external voltage spike)
   - Output pin driven beyond VDD or below GND
   - Power supply transient (VDD bounce)
   - Radiation (single-event latch-up in space applications)

2. Positive feedback loop:
   - Current flows through Rsub → VBE of Q2 rises above ~0.7V
   - Q2 turns ON → collector current flows into n-well
   - Current through Rwell → VEB of Q1 rises above ~0.7V
   - Q1 turns ON → collector current flows into p-substrate
   - This feeds back into Q2's base → REGENERATIVE feedback

3. Latch-up sustained when:
   β_pnp × β_npn × (Rwell × Rsub product term) > 1
   (Barkhausen criterion for positive feedback)

4. Result: Low-impedance path from VDD to GND
   Current can be > 100 mA → thermal destruction
```

### 6.3 Latch-Up Prevention

```
1. Guard Rings:
   - P+ guard ring around NMOS (tied to GND): collects minority carriers
     from substrate, reduces Rsub
   - N+ guard ring around PMOS (tied to VDD): collects minority carriers
     from n-well, reduces Rwell
   - Reduces β of parasitic BJTs and lowers well/substrate resistance

2. Increase spacing (NMOS-to-PMOS):
   - Larger separation → weaker parasitic BJT coupling
   - DRC rule: minimum N-well to P+ diffusion spacing

3. Retrograde well:
   - Higher doping deeper in well → lower Rwell
   - Reduces BJT gain

4. Epitaxial substrate:
   - Thin lightly-doped epi layer on heavily-doped substrate
   - Heavy doping reduces Rsub dramatically
   - Most effective single prevention technique

5. Trench isolation (STI/DTI):
   - Shallow Trench Isolation between devices
   - Deep Trench Isolation (DTI) for complete well isolation
   - Physically breaks parasitic BJT current paths

6. SOI (Silicon-On-Insulator):
   - Buried oxide completely isolates devices
   - Eliminates latch-up entirely
   - Used in high-reliability and some high-performance processes

7. Layout rules:
   - Every well must have a well tap (contact to VDD/GND)
   - Maximum distance from any transistor to nearest well tap
   - Typical rule: well tap every 10-20 μm
```

### 6.4 Latch-Up Testing

```
JEDEC JESD78 standard:
  - Positive and negative current injection test (±100 mA)
  - Positive and negative voltage overstress (VDD + 1.5V)
  - At elevated temperature (typically 125°C)

Pass criteria:
  - No latch-up (current returns to normal after trigger removed)
  - Supply current < specified limit during test
  - Device functional after test
```

---

## 7. ESD Protection

### 7.1 ESD Events

```
Human Body Model (HBM):
  - Models discharge from human finger
  - 100 pF charged to 2-4 kV, discharged through 1.5 kΩ
  - Peak current: ~1.3 A, duration: ~150 ns
  - Industry target: survive ±2 kV HBM

Charged Device Model (CDM):
  - IC itself becomes charged, then discharges through one pin
  - Very fast (< 1 ns), high peak current (> 10 A)
  - More damaging to thin gate oxides than HBM
  - Industry target: survive ±500 V CDM

Machine Model (MM):
  - Models discharge from automated equipment
  - 200 pF, 0 Ω (no series resistance)
  - Largely replaced by CDM in modern standards
```

### 7.2 ESD Protection Circuits

```
Basic I/O pad ESD protection:

         VDD ─────────────────────────
                    |
                 ┌──┤ D1 (diode to VDD)
                 │  |
  PAD ───────────┤  ├── Internal circuit
                 │  |
                 └──┤ D2 (diode to GND)
                    |
         GND ─────────────────────────

  Positive ESD on PAD: current flows through D1 to VDD
  Negative ESD on PAD: current flows through D2 from GND

Additional elements:
  - Power clamp between VDD and GND (RC-triggered NMOS)
  - Secondary protection (series resistor + smaller clamp closer to gate)
```

```
Grounded-Gate NMOS (ggNMOS) clamp:
  - Large NMOS with gate/source/body tied to GND
  - Triggers via snapback: drain voltage exceeds BV → avalanche
    → substrate current → parasitic NPN turns on → low-impedance path
  - Holding voltage ~3-5V, can clamp large currents

  Trigger voltage (Vt1) > VDD (so it doesn't activate during normal operation)
  Holding voltage (Vh) > VDD (to avoid latch-up after ESD event)
```

### 7.3 ESD Design Rules

```
1. All I/O pads must have primary ESD protection
2. All power domains need power clamps
3. No gate oxide directly connected to I/O pad without protection
4. CDM protection for all cross-domain signals
5. Antenna rules: long metal connected to gate during fabrication
   can accumulate charge → gate oxide damage (same mechanism as ESD)
```

---

## 8. FinFET and Advanced Nodes

### 8.1 Why FinFET?

```
Planar MOSFET below 22nm:
  - Gate loses control of channel (short-channel effects dominate)
  - Subthreshold slope >> 60 mV/dec ideal
  - DIBL > 100 mV/V
  - Leakage current unacceptably high

FinFET solution:
  - Wrap gate around 3 sides of a thin "fin" of silicon
  - Much better electrostatic control
  - Subthreshold slope ≈ 65-70 mV/dec (near ideal)
  - DIBL < 50 mV/V
  - Dramatically reduced leakage
```

### 8.2 FinFET Structure

```
  Top view:                    Cross-section (perpendicular to fin):

                                       Gate
  ←── Fin direction ──→                 |
                                   ┌────┤────┐
  S ═══════════════════ D          │    │    │  Gate wraps
       │  │  │  │  │               │  ┌─┴─┐ │  3 sides
       Gate contacts               │  │Fin│ │
       (perpendicular              │  │   │ │
        to fin)                    │  └───┘ │
                                   └────────┘
                                     BOX/Substrate

  Weff = 2*Hfin + Wfin   (effective width = 2 sides + top)
  Typical 7nm: Hfin ≈ 50nm, Wfin ≈ 7nm → Weff ≈ 107nm per fin
```

### 8.3 Fin Quantization

```
CRITICAL DIFFERENCE from planar MOSFET:

Planar: W is continuous — designer can choose any W (e.g., 120nm, 135nm)
FinFET: W is QUANTIZED — width = N_fins × Weff_per_fin

  Example (7nm):
    1 fin:  Weff = 107nm
    2 fins: Weff = 214nm
    3 fins: Weff = 321nm
    ...
    No intermediate values possible!

Impact on design:
  - Cannot fine-tune sizing like in planar (must use integer fins)
  - Minimum sizing = 1 fin (vs minimum W in planar)
  - Drive strength ratios are coarser (1:2:3:4... vs continuous)
  - Library cells: INV_X1 = 1 fin, INV_X2 = 2 fins, etc.
  - Makes analog design harder (less precision in current mirrors)
```

### 8.4 GAAFET (Gate-All-Around) — Beyond FinFET

```
3nm and below: GAA / Nanosheet FET
  - Gate wraps ALL 4 sides of the channel
  - Better electrostatic control than FinFET
  - Channel = stacked nanosheets (multiple horizontal sheets)
  - Width tunable by nanosheet width (more flexibility than FinFET)

  Samsung 3nm GAA: 2022 (first production GAAFET)
  Intel RibbonFET: Intel 20A (2024)
  TSMC N2: GAA nanosheet (2025)

     Gate          Gate
    ┌────┐        ┌────┐
    │Sheet│        │Sheet│   Multiple stacked nanosheets
    └────┘        └────┘
    ┌────┐        ┌────┐
    │Sheet│        │Sheet│
    └────┘        └────┘
     S              D
```

### 8.5 Back-End-of-Line (BEOL) Scaling

```
As transistors shrink, metal interconnects become the bottleneck:

  Wire resistance: R = ρ * L / (W * H)
    At 7nm: Cu resistivity increases due to grain boundary and
    surface scattering (ρeff >> ρbulk when wire width ≈ electron mean free path)

  Wire capacitance: C = ε * L * H / S  (parallel plate between adjacent wires)
    Spacing S shrinks → C increases → crosstalk worsens

  RC delay of interconnect:
    τ = R × C ∝ ρ * ε * L² / (W * H * S)
    Doubling wire length → 4× delay (quadratic!)

Solutions:
  - Low-k dielectrics: reduce ε (k < 3.0, air gaps for ultra-low-k)
  - Alternative metals: Co, Ru for narrow lines (better than Cu at < 20nm pitch)
  - Repeater insertion: break long wires with buffers
  - Metal layer count: 12-15+ layers at advanced nodes
```

---

## 9. Process Variations and DTCO

### 9.1 Types of Variation

```
Systematic variation:
  - Predictable, depends on layout context
  - Lithography: line-end shortening, corner rounding
  - CMP (Chemical Mechanical Polishing): metal thickness varies with density
  - Well proximity effect: Vth varies near well boundary

Random variation:
  - Unpredictable, follows statistical distributions
  - Random Dopant Fluctuation (RDF): discrete dopant atoms cause Vth variation
    σ(Vth) ∝ 1/√(W*L)  → worse for smaller transistors
  - Line Edge Roughness (LER): random variation in gate length
  - Oxide thickness variation (Tox)

Within-die (WID) vs Die-to-die (D2D):
  WID: transistors on the same chip differ from each other
  D2D: average parameters differ between chips
  Both must be accounted for in timing analysis (OCV, AOCV, POCV)
```

### 9.2 Process Corners

```
Traditional corners:
  TT: Typical NMOS, Typical PMOS (nominal)
  FF: Fast NMOS, Fast PMOS (low Vth, high μ)
  SS: Slow NMOS, Slow PMOS (high Vth, low μ)
  FS: Fast NMOS, Slow PMOS (skewed)
  SF: Slow NMOS, Fast PMOS (skewed)

Combined with voltage and temperature:
  Best-case speed: FF, high VDD, low temperature (0°C or -40°C)
  Worst-case speed: SS, low VDD, high temperature (125°C)

  EXCEPTION at advanced nodes — temperature inversion:
    Below ~0.8V, lower temperature → SLOWER (Vth increases more than μ improves)
    This reverses the traditional hot=slow assumption!
```

### 9.3 DTCO (Design-Technology Co-Optimization)

```
DTCO: Simultaneously optimizing the process technology and the design methodology
to achieve the best PPA (Power, Performance, Area).

Examples:
  - Standard cell height reduction:
    7.5T → 6.5T → 6T → 5T track height
    Fewer metal tracks → smaller cells → higher density
    But: fewer routing tracks → higher congestion

  - Fin depopulation:
    Selectively remove fins in non-critical cells → save power
    Requires tight collaboration between design and process teams

  - CPODE (Continuous Poly on Diffusion Edge):
    Cut the poly on the diffusion edge to isolate transistors
    Reduces cell-to-cell spacing → higher density

  - Buried Power Rail (BPR):
    Move VDD/VSS rails below the transistor layer
    Frees up M1 routing tracks → more signal routing
    Planned for 2nm and beyond

  - Backside Power Delivery Network (BSPDN):
    Power delivered from the backside of the wafer
    Completely separates power and signal routing
    Intel PowerVia, TSMC N2P
```

---

## 10. Interview Q&A

**Q1: Draw the VTC of a CMOS inverter and label all five regions of operation.**

(See Section 2.2 above.) Region A: NMOS off, PMOS linear, Vout=VDD. Region B: NMOS
saturation, PMOS linear. Region C: both saturation (transition). Region D: NMOS linear,
PMOS saturation. Region E: NMOS linear, PMOS off, Vout=0.

**Q2: Derive the switching threshold VM. How do you make VM = VDD/2?**

Set IDn = IDp with both in saturation: kn(VM-Vthn)² = kp(VDD-VM-|Vthp|)². Solving gives
VM = (Vthn + r(VDD-|Vthp|))/(1+r) where r = √(kp/kn). For VM = VDD/2 with equal
thresholds: kp = kn, meaning Wp/Lp ≈ 2.5 × Wn/Ln (to compensate for lower hole mobility).

**Q3: Why is NAND preferred over NOR in CMOS?**

NAND has NMOS in series and PMOS in parallel. NOR has PMOS in series and NMOS in parallel.
Since PMOS is ~2.5× slower than NMOS (lower mobility), series PMOS in NOR creates very
high pull-up resistance, making NOR gates much slower and larger for equal drive strength.
For an N-input NOR, each PMOS must be N×2.5× minimum width — prohibitively large.

**Q4: Explain latch-up. How is it triggered and prevented?**

Latch-up occurs when parasitic PNP and NPN BJTs in CMOS form a positive feedback loop
(thyristor/SCR structure). Triggered when substrate or well current forward-biases one
BJT, which then feeds the other. Once triggered, a low-impedance VDD-to-GND path forms
with potentially destructive current. Prevention: guard rings (reduce well/substrate
resistance), sufficient NMOS-PMOS spacing, epitaxial substrate, trench isolation, frequent
well taps, and SOI processes (which eliminate latch-up entirely).

**Q5: What is the body effect and when does it matter?**

The body effect increases Vth when source-to-body voltage (VSB) is non-zero:
Vth = Vth0 + γ(√(2φF+VSB) - √(2φF)). It matters in stacked transistors (e.g., 4-input
NAND: the NMOS closest to the output has its source above GND due to other NMOS below it,
increasing VSB and thus Vth, which slows the gate). Also matters in source-follower
circuits and transmission gates.

**Q6: Compare planar MOSFET, FinFET, and GAAFET.**

Planar: gate contacts channel from one side. Good to ~28nm. Poor short-channel control
below 22nm. FinFET: gate wraps 3 sides of a vertical fin. Used at 22nm-5nm. Quantized
width (integer number of fins). Excellent short-channel control. GAAFET: gate wraps all
4 sides of horizontal nanosheets. Used at 3nm and below. Even better electrostatic control.
Width somewhat adjustable via nanosheet width.

**Q7: What is fin quantization and how does it impact design?**

In FinFET, transistor width is quantized — only integer numbers of fins are possible.
Unlike planar CMOS where W can be any continuous value, FinFET designs must choose 1, 2,
3... fins. This makes fine-grained sizing impossible, affects drive strength ratios in
standard cells, complicates analog design (current mirrors need precise ratios), and
requires library architects to carefully choose fin counts for each cell variant.

**Q8: Explain the temperature inversion effect at advanced nodes.**

At high VDD (>0.9V), the traditional relationship holds: higher temperature → slower
(mobility decreases). But at low VDD (<0.8V), higher temperature → FASTER. This is because
Vth decreases with temperature, and at low VDD the Vth reduction provides more
current increase than the mobility decrease causes current loss. The crossover voltage
where temperature has no effect is called the zero-temperature-coefficient (ZTC) point.

**Q9: What are noise margins? How do you calculate them?**

NMH = VOH - VIH (tolerance for high logic level). NML = VIL - VOL (tolerance for low
level). VIH and VIL are defined as the points on the VTC where gain = -1. For a symmetric
CMOS inverter with VDD = 0.7V and Vt = 0.3V, NMH = NML ≈ 0.34V (about 48% of VDD).
Ideal CMOS has VOH = VDD and VOL = 0, giving excellent noise margins.

**Q10: What is velocity saturation and why does it matter?**

In long-channel MOSFETs, current scales quadratically with (VGS-Vth). But in
short-channel devices (< 100nm), the lateral electric field is so high that carrier
velocity saturates at vsat ≈ 10^7 cm/s. This makes IDS proportional to (VGS-Vth)
linearly instead of quadratically, reducing the benefit of higher VGS. It also means
that NMOS and PMOS performance is closer than predicted by mobility ratio alone (since
both saturate at similar velocities).

**Q11: What is DIBL and how does it affect timing?**

Drain-Induced Barrier Lowering: the drain voltage reduces the source-channel potential
barrier, effectively lowering Vth. Higher VDS → lower Vth → more current → faster
switching. But also more leakage (lower Vth at VDS = VDD). DIBL creates a coupling
between neighboring gates through shared drain nodes and is a major concern for timing
variation at advanced nodes (η can be 50-100 mV/V at 7nm).

**Q12: Compare HBM and CDM ESD models.**

HBM models a human touching a pin: 100pF through 1.5kΩ, peak ~1.3A over ~150ns. CDM
models the IC itself discharging: very fast (<1ns), peak >10A. CDM is more damaging to
thin gate oxides because the high peak current density causes localized oxide breakdown.
Modern specs typically require ±2kV HBM and ±500V CDM survival.

**Q13: Why does wire resistance increase at advanced nodes?**

At 7nm and below, Cu wire width approaches the electron mean free path (~40nm). Two
effects increase resistivity: (1) Surface scattering — electrons bounce off wire surfaces.
(2) Grain boundary scattering — more grain boundaries per unit length. The effective
resistivity can be 2-5× higher than bulk Cu. This is why alternative metals (Co, Ru)
are being explored — they have shorter mean free paths so they're less affected by
narrow widths.

**Q14: What is the subthreshold slope and why can't it go below 60 mV/dec?**

The subthreshold slope S = (kT/q) × ln(10) × (1 + Cd/Cox) defines how sharply the
transistor turns off. The theoretical minimum at room temperature is (kT/q) × ln(10) ≈
60 mV/decade (when Cd/Cox → 0, i.e., perfect gate control). This is the Boltzmann tyranny
— set by thermal physics. It limits how low VDD can go while maintaining adequate
on/off ratio. Overcoming this requires non-classical devices (tunnel FET, negative
capacitance FET).

**Q15: What is the difference between static and dynamic CMOS logic?**

Static CMOS: pull-up (PMOS) and pull-down (NMOS) networks. Outputs are always driven.
Ratioless, full rail-to-rail swing, good noise margins, but 2N transistors for N inputs.
Dynamic CMOS: uses precharge/evaluate phases with a clock. N+2 transistors for N inputs,
faster (lower input capacitance since no PMOS in logic network), but sensitive to charge
sharing, noise, clock skew, and only supports non-inverting functions (in domino).

**Q16: What is charge sharing in dynamic logic and how do you fix it?**

During evaluation, internal nodes in the NMOS pull-down network may not be precharged.
When evaluation begins, charge from the precharged output node redistributes to these
internal nodes, causing the output voltage to drop even when the PDN shouldn't conduct.
Fix: precharge all internal nodes (add PMOS to each internal node), or add a keeper
(weak PMOS from output to VDD, feedback-controlled) that fights the charge redistribution.

**Q17: What are process corners and why do we need MCMM?**

Process corners (TT, FF, SS, FS, SF) model manufacturing variation in NMOS and PMOS
parameters. MCMM (Multi-Corner Multi-Mode) runs timing analysis at multiple
corner-mode combinations simultaneously: e.g., SS/0.9*VDD/125°C for setup,
FF/1.1*VDD/-40°C for hold. This is needed because setup violations worsen at slow
corners while hold violations worsen at fast corners. Modern tools (PrimeTime, Tempus)
handle 20+ MCMM scenarios in a single analysis.

**Q18: Explain the concept of logical effort.**

Logical effort quantifies the delay cost of computing a logic function compared to an
inverter. It's defined as the ratio of input capacitance of a gate to that of an
inverter with equal output drive. For minimum delay through a path, each stage should
have equal stage effort (product of logical effort, electrical effort, and branching
effort). This leads to optimal gate sizing: larger gates for higher fan-out stages.

**Q19: What is random dopant fluctuation (RDF)?**

In modern transistors, the channel is so small that individual dopant atoms matter.
A minimum-size 7nm transistor might have only 10-20 dopant atoms under the gate.
The exact number and position of these atoms is random, causing Vth variation:
σ(Vth) ∝ 1/√(W×L). This is a major source of mismatch and timing variation at
advanced nodes. FinFETs partially mitigate RDF by using undoped channels with
workfunction engineering to set Vth.

**Q20: What is CPODE and why is it important?**

Continuous Poly on Diffusion Edge cuts the polysilicon gate at the boundary of active
region to isolate adjacent transistors. This allows tighter cell-to-cell spacing compared
to dummy poly approach, increasing standard cell density. It's a key enabler for smaller
standard cell heights (6T, 5.5T) at 5nm and below.

**Q21: Explain buried power rail (BPR) and backside power delivery.**

Buried Power Rail places VDD/VSS rails below the transistor layer (buried in the silicon),
freeing up Metal 1 for signal routing. Backside Power Delivery (BSPDN) takes this further
— the entire power grid is on the back of the wafer, completely decoupling power and signal
routing. Benefits: more routing resources, lower IR drop (shorter power paths), better cell
density. Intel PowerVia and TSMC N2P are early implementations.

**Q22: Why does lowering VDD help power more than it hurts performance?**

Dynamic power ∝ VDD². Delay ∝ VDD/(VDD-Vth)^α where α ≈ 1-2. So a 10% VDD reduction
gives ~19% power savings but only ~10-15% delay increase (when VDD >> Vth). The
energy-delay product (EDP) improves with VDD reduction until VDD approaches ~3nVT
(near-threshold). This is why DVFS is so effective — even modest voltage reduction
yields significant power savings with manageable performance loss.

**Q23: What is antenna effect and how is it fixed?**

During metal etching in fabrication, long metal lines connected to a gate can accumulate
charge from the plasma. This charge can damage the thin gate oxide via Fowler-Nordheim
tunneling. The antenna ratio = metal_area / gate_area. If it exceeds the process limit,
fixes include: adding a diode to the gate node (provides discharge path), breaking the
long metal into segments on different layers (layer hopping), or rerouting.

**Q24: What is the difference between SOI and bulk CMOS?**

In bulk CMOS, transistors are built directly on the silicon substrate — they share the
substrate and have body ties, parasitic capacitance, body effect, and latch-up risk.
In SOI, a buried oxide (BOX) layer isolates each transistor from the substrate.
Benefits: no latch-up, lower junction capacitance (30-50% less), reduced body effect,
better short-channel control. Drawbacks: floating body effects (in partially-depleted SOI),
self-heating (oxide is a thermal insulator), higher wafer cost.

**Q25: How does CMP affect ASIC design?**

Chemical Mechanical Polishing planarizes metal and dielectric surfaces. If metal density
is non-uniform, CMP causes thickness variation — dense regions polish faster (dishing),
sparse regions stay thick. This affects: wire resistance (thinner wire = higher R),
capacitance, and timing. Design rules require metal density to stay within bounds
(typically 20-80%), achieved by inserting dummy metal fill in sparse regions.
