# CMOS Fundamentals and Device Physics — Senior Engineer Deep Dive

## Table of Contents
1. MOSFET Operation
2. CMOS Inverter and VTC
3. Noise Margins
4. Propagation Delay and Power-Delay Product
5. CMOS Logic Families & I/O Signaling
6. Latch-Up
7. ESD Protection
8. FinFET and Advanced Nodes
9. Process Variations and DTCO
10. Elmore Delay Model
11. Logical Effort
12. 6T SRAM Cell
13. Leakage Current Breakdown
14. Numbers to Memorize

---

## 1. MOSFET Operation

### 1.1 NMOS Transistor — Regions of Operation

```ascii-graph
                  Gate (G)
                   |
            ───────┤ oxide (SiO2, tox)
           |       |       |
    Source (S)    body    Drain (D)
      n+     p-substrate    n+
```

**Three operating regions (NMOS, VGS > 0, VDS > 0):**

```verilog
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
```text
μn:    Electron mobility (~400 cm²/V·s for bulk Si at 300K)
Cox:   Gate oxide capacitance = εox / tox
W/L:   Width-to-length ratio (designer's knob)
Vth:   Threshold voltage (~0.3-0.5V for modern processes)
λ:     Channel-length modulation (~0.05-0.1 V⁻¹)
```

### 1.2 PMOS Transistor

Same equations but with opposite signs/polarities:
```ascii-graph
Cutoff:      VGS > Vth (Vth < 0 for PMOS, so |VGS| < |Vth|)
Linear:      |VGS| > |Vth|, |VDS| < |VGS| - |Vth|
Saturation:  |VGS| > |Vth|, |VDS| ≥ |VGS| - |Vth|

Hole mobility μp ≈ 150-200 cm²/V·s (about 2-3x lower than electrons)
→ PMOS must be 2-3x wider than NMOS for equal drive strength
```

### 1.3 Threshold Voltage

```text
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

```ascii-graph
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

```ascii-graph
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

```ascii-graph
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

```ascii-graph
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

```text
(1/2) * kn * (VM - Vthn)² = (1/2) * kp * (VDD - VM - |Vthp|)²

where kn = μn*Cox*(Wn/Ln), kp = μp*Cox*(Wp/Lp)

Let r = √(kp/kn):

VM = (Vthn + r*(VDD - |Vthp|)) / (1 + r)
```

**For symmetric VTC (VM = VDD/2):**
```ascii-graph
kp/kn = 1  →  (μp*Wp/Lp) = (μn*Wn/Ln)
Since μp ≈ μn/2.5:  Wp/Lp ≈ 2.5 * (Wn/Ln)
```

**Numerical example (7nm):**
```text
VDD = 0.7V, Vthn = 0.3V, |Vthp| = 0.3V
r = 1 (symmetric sizing):
VM = (0.3 + 1*(0.7-0.3)) / (1+1) = 0.7/2 = 0.35V = VDD/2  ✓
```

---

## 3. Noise Margins

### 3.1 Definition

VOH: Output high voltage (Vout when Vin = 0) = VDD
VOL: Output low voltage (Vout when Vin = VDD) = 0
VIH: Input high voltage (min Vin recognized as logic 1)
VIL: Input low voltage (max Vin recognized as logic 0)

VIH and VIL are defined as the points where |dVout/dVin| = 1
(unity gain points on the VTC)

- **NMH (Noise Margin High)** = `VOH - VIH`
- **NML (Noise Margin Low)** = `VIL - VOL`

```ascii-graph
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

```ascii-graph
VIL ≈ (2*Vout + Vin - Vt) / (solving dVout/dVin = -1)

For symmetric inverter (long-channel approximation, from region-by-region VTC analysis):
  VIL ≈ (3*VDD + 2*Vt) / 8
  VIH ≈ (5*VDD - 2*Vt) / 8

  NMH = VDD - VIH = VDD - (5*VDD - 2*Vt)/8 = (3*VDD + 2*Vt)/8
  NML = VIL - 0   = (3*VDD + 2*Vt)/8

  NMH = NML → symmetric noise margins
```

**Numerical example:**
- **VDD** = `0.7V, Vt = 0.3V:`
   - NMH = NML = (3*0.7 + 2*0.3)/8 = (2.1 + 0.6)/8 = 0.3375V

NM/VDD = 0.3375/0.7 = 48.2% of VDD

This is high — ideal CMOS has excellent noise margins
Compare with: NMOS-only logic NM ≈ 20-30% of VDD

### 3.3 Factors That Degrade Noise Margins

```ascii-graph
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

|Gain| = |dVout/dVin| at VM

**For long-channel CMOS:**
   - |Gain| = (kn + kp) * (VDD/2 - Vt) / (λn + λp) * VDD

Typically |Gain| > 10-50

This means: a signal degraded by noise gets "cleaned up" after
passing through a few inverters → digital logic is noise-tolerant

---

## 4. Propagation Delay and Power-Delay Product

### 4.1 Propagation Delay

```text
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
```ascii-graph
Rn = Rp → kn(VDD - Vthn)² = kp(VDD - |Vthp|)²
For Vthn = |Vthp|: kn = kp → Wp ≈ 2.5 * Wn (same as symmetric VM)
```

### 4.2 Delay Optimization

```ascii-graph
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

```ascii-graph
PDP = P_avg × tp
    = (α * CL * VDD² * f) × (k * CL * VDD / ID)
    ∝ CL * VDD²    (energy per transition)

EDP = PDP × tp  ∝ CL² * VDD³ / ID
    → Minimize EDP by finding optimal VDD (below VDD_nominal)

At optimal VDD for minimum EDP:
  VDD_opt ≈ 3 * n * VT ≈ 0.2-0.3V (near-threshold computing)
```

---

## 5. CMOS Logic Families & I/O Signaling

### 5.1 Static CMOS (Complementary)

```ascii-graph
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
```ascii-graph
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

```ascii-graph
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

```ascii-graph
         A ──┤├── B         NMOS
         A ──┤├── B         PMOS (complementary control)
              |
Combined: passes both 0 and 1 without threshold drop

Used in: MUXes, XOR, latches
Advantage: compact, good for pass logic
Disadvantage: no gain → signal degrades through chain of TGs
```

### 5.4 Dynamic Logic (Domino)

```ascii-graph
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
```ascii-graph
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

### 5.5 I/O Signaling Standards (Off-Chip)

§5.1–5.4 are *on-chip* logic styles driving fF-scale gate loads. **I/O cells (pads)** are a different world: they drive pF-scale off-chip loads (package + PCB trace + connector + receiver), so they trade speed/area for **drive strength, slew-rate control, defined voltage levels, termination, and ESD**. The signaling *standard* is the TX↔RX contract on the board; pick it from three families.

**(1) Rail-referenced single-ended** — receiver compares to fixed thresholds referenced to its own rails. Cheapest (1 pin/signal), but noise margin is VDDQ-bounded and rate is limited by ground bounce / SSO noise.

| Standard | VDDQ | Thresholds | Typical use |
|----------|------|-----------|-------------|
| LVTTL | 3.3 V | VIH 2.0 / VIL 0.8 V (TTL-compatible) | legacy GPIO |
| LVCMOS | 3.3 / 2.5 / 1.8 / 1.5 / 1.2 V | VIH≈0.7·VDDQ, VIL≈0.3·VDDQ (rail-to-rail) | GPIO, SPI, UART, JTAG, I²C |

**(2) VREF-referenced, terminated single-ended** — for wide, fast parallel buses (DRAM). Receiver compares to an external **VREF ≈ VDDQ/2** and uses **on-die termination (ODT)** to kill reflections, so the swing can be small (fast edges).

| Standard | Reference / termination | Used by |
|----------|------------------------|---------|
| SSTL (Stub-Series Terminated Logic) — SSTL_18/15/135 | VTT = VDDQ/2 series-stub termination | DDR2 / DDR3 / DDR3L |
| HSTL (High-speed Transceiver Logic) | VTT termination | QDR SRAM, older FPGA banks |
| POD (Pseudo-Open-Drain) — POD12/135 | terminate to **VDDQ** (only a "0" sinks DC) | DDR4 / DDR5, GDDR |

**(3) Differential** — two complementary wires; receiver senses the *difference*, so common-mode noise cancels → small swing, high speed, low EMI, at 2 pins/signal.

| Standard | Swing / Vcm | Termination | Used by |
|----------|-------------|-------------|---------|
| LVDS | ~350 mV diff, Vcm ≈ 1.2 V | 100 Ω across the pair | display/sensor links, moderate Gb/s |
| CML (current-mode logic) | ~400–800 mV | 50 Ω/leg to VDD | SerDes — PCIe / Ethernet / HBM PHY, multi-Gb/s |

(DDR clock/strobe DQS also use *differential* SSTL/HSTL.)

**Tradeoffs (interview):**
- **Single-ended vs differential:** SE is pin-efficient and cheap, but noise margin and max rate are limited by SSO/ground bounce; differential rejects common-mode, runs Gb/s+, and its small swing cuts dynamic I/O power — at 2× the pins.
- **Why VREF + ODT for DRAM:** a wide bus at high rate can't tolerate full-swing reflections; centering on VREF with ODT yields clean eyes at low swing.
- **SSTL → POD (DDR3→DDR4/5):** POD terminates to VDDQ, so a bus that idles high burns termination DC only on "0"s → lower I/O power as data rates climbed.
- **Drive strength & slew:** a stronger driver gives faster edges but more SSO/overshoot/EMI; I/O cells expose selectable drive strength and slew-rate control to trade SI for speed.
- **Open-drain (I²C/SMBus):** wired-AND with an external pull-up — bidirectional on one wire, but slow (RC pull-up).

Related: noise-margin basis (§3 above); reflections/termination + SerDes equalization → [Signal Integrity](../05_Backend_Physical_Design/02_Signal_Integrity_Reliability.md); DDR I/O & ODT → [DDR Controller](../01_Architecture_and_PPA/10_DDR_Controller.md); package-level high-speed I/O → [IC Packaging](../07_Manufacturing_and_Bringup/02_IC_Packaging.md).

---

## 6. Latch-Up

### 6.1 The Parasitic PNPN Structure

In a CMOS inverter, the n-well (for PMOS) and p-substrate (for NMOS) create a parasitic
thyristor (SCR = Silicon Controlled Rectifier):

```ascii-graph
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

```ascii-graph
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

```ascii-graph
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

```ascii-graph
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

JEDEC JESD78 standard:
- Positive and negative current injection test (±100 mA)
- Positive and negative voltage overstress (VDD + 1.5V)
- At elevated temperature (typically 125°C)

1. Pass criteria:
- No latch-up (current returns to normal after trigger removed)
- Supply current < specified limit during test
- Device functional after test

---

## 7. ESD Protection

### 7.1 ESD Events

```text
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

```ascii-graph
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

```ascii-graph
Grounded-Gate NMOS (ggNMOS) clamp:
  - Large NMOS with gate/source/body tied to GND
  - Triggers via snapback: drain voltage exceeds BV → avalanche
    → substrate current → parasitic NPN turns on → low-impedance path
  - Holding voltage ~3-5V, can clamp large currents

  Trigger voltage (Vt1) > VDD (so it doesn't activate during normal operation)
  Holding voltage (Vh) > VDD (to avoid latch-up after ESD event)
```

### 7.3 ESD Design Rules

1. All I/O pads must have primary ESD protection
2. All power domains need power clamps
3. No gate oxide directly connected to I/O pad without protection
4. CDM protection for all cross-domain signals
5. Antenna rules: long metal connected to gate during fabrication can accumulate charge → gate oxide damage (same mechanism as ESD)

---

## 8. FinFET and Advanced Nodes

### 8.1 Why FinFET?

```text
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

```ascii-graph
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

```text
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

```ascii-graph
3nm and below: GAA / Nanosheet FET
  - Gate wraps ALL 4 sides of the channel
  - Better electrostatic control than FinFET
  - Channel = stacked nanosheets (multiple horizontal sheets)
  - Width tunable by nanosheet width (more flexibility than FinFET)

  Samsung 3nm GAA (MBCFET): 2022 (first production GAAFET)
    - Multi-Bridge Channel FET — nanosheets connected in parallel
    - Production since July 2022, used in Samsung Exynos and Qualcomm
  TSMC N2: GAA nanosheet, production 2025-2026
    - ~15% speed improvement or ~30% power reduction over N3
    - First TSMC node to use GAA, replacing FinFET after N3/N3E
  Intel 18A (1.8nm): RibbonFET + PowerVia, production 2025
    - Intel cancelled 20A node and moved directly to 18A
    - RibbonFET = Intel's name for GAA nanosheet
    - Combined with PowerVia backside power delivery (see Section 9.3)

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

As transistors shrink, metal interconnects become the bottleneck:

Wire resistance: R = ρ * L / (W * H)
At 7nm: Cu resistivity increases due to grain boundary and
surface scattering (ρeff >> ρbulk when wire width ≈ electron mean free path)

Wire capacitance: C = ε * L * H / S  (parallel plate between adjacent wires)
Spacing S shrinks → C increases → crosstalk worsens

RC delay of interconnect:
τ = R × C ∝ ρ * ε * L² / (W * H * S)
Doubling wire length → 4× delay (quadratic!)

**Solutions:**
   - Low-k dielectrics: reduce ε (k < 3.0, air gaps for ultra-low-k)
   - Alternative metals: Co, Ru for narrow lines (better than Cu at < 20nm pitch)
   - Repeater insertion: break long wires with buffers
   - Metal layer count: 12-15+ layers at advanced nodes

---

## 9. Process Variations and DTCO

### 9.1 Types of Variation

**Systematic variation:**
   - Predictable, depends on layout context
   - Lithography: line-end shortening, corner rounding
   - CMP (Chemical Mechanical Polishing): metal thickness varies with density
   - Well proximity effect: Vth varies near well boundary

**Random variation:**
   - Unpredictable, follows statistical distributions
   - Random Dopant Fluctuation (RDF): discrete dopant atoms cause Vth variation
   - σ(Vth) ∝ 1/√(W*L)  → worse for smaller transistors
   - Line Edge Roughness (LER): random variation in gate length
   - Oxide thickness variation (Tox)

**Within-die (WID) vs Die-to-die (D2D):**
   - WID: transistors on the same chip differ from each other
   - D2D: average parameters differ between chips
   - Both must be accounted for in timing analysis (OCV, AOCV, POCV)

### 9.2 Process Corners

```ascii-graph
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

```ascii-graph
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
    Intel PowerVia (first in Intel 18A, 2025):
      - VDD/VSS delivered through backside vias (TSV-like)
      - Eliminates IR drop on frontside metal (up to 50% reduction)
      - Frees M0/M1 for signal routing → higher cell density
      - Requires wafer thinning to expose backside
    TSMC N2P (2026): Super Power Rail (SPR) backside delivery
    Samsung SF2 (2025): BSPDN on 2nm node
```

### 9.4 GAA Nanosheet Channel Width Modulation

Key advantage of GAA over FinFET: continuous width control

FinFET width: W = N_fins * (2*H + W_fin), quantized in units of one fin
GAA nanosheet: W = N_sheets * W_sheet, where W_sheet is continuously tunable

Example: 3 stacked nanosheets, each 30nm wide
Weff = 4 * 3 * 30nm = 360nm (gate wraps all 4 sides of each sheet)
Can also make W_sheet = 25nm for Weff = 300nm
Or W_sheet = 20nm for Weff = 240nm → continuous adjustment!

Sheet width range: ~15nm to ~50nm per sheet (technology-dependent)
Number of sheets: typically 3-5 stacked vertically

**Implications for standard cell design:**
   - Can tune drive strength more precisely than FinFET
   - Library cells can have optimized widths, not just integer multiples
   - Analog/mixed-signal benefits: better current mirror matching
   - But: wider sheets → more gate capacitance (trade-off)

### 9.5 Near-Threshold Computing

```verilog
Operating transistors near Vth (VDD ≈ 0.4-0.6V) for extreme energy efficiency:

  Energy per operation ∝ VDD² (dominated by dynamic power)
  At VDD ≈ 3*n*VT ≈ 0.3-0.4V: optimal energy-delay product

  Benefits:
    - 5-10x energy reduction vs. nominal VDD
    - Still reasonable performance (50-70% of max frequency)

  Challenges:
    - Exponential leakage sensitivity to Vth variation
    - Subthreshold slope limits on/off ratio at low VDD
    - SRAM stability worst at low VDD (read/write margin collapse)
    - Performance variability increases dramatically

  Applications:
    - IoT sensors (energy-harvested devices)
    - Always-on edge AI (wake-word detection)
    - Ultra-low-power microcontrollers (ARM Cortex-M0+ at 0.5V)

  Design techniques for near-threshold:
    - Separate VDD domains: logic at VDD_low, SRAM at VDD_nominal
    - Adaptive body bias to compensate Vth variation
    - Replica circuits for dynamic VDD/frequency adjustment
```

---

## 10. Elmore Delay Model

### 11.1 RC Tree Delay

For a series of $N$ inverters each driving a wire segment with resistance $R_w$ and
capacitance $C_w$, plus gate capacitance $C_g$, the Elmore delay through the chain is:

$$t_{\text{Elmore}} = \sum_{i=1}^{N} R_i \cdot C_{\text{downstream},i}$$

where $C_{\text{downstream},i}$ is the total capacitance seen from node $i$ to ground
through all downstream paths. The Elmore model treats each wire segment as a lumped RC
element and sums the product of each resistance with all capacitance it drives — a
first-order upper bound on the 50% step-response delay.

### 11.2 Single Inverter with Wire Load

For an inverter with on-resistance $R_p$ driving a wire of length $L$ (per-unit-length
$r$ and $c$) into a load capacitance $C_L$:

$$t_{\text{Elmore}} = 0.38 \cdot R_p \cdot C_{\text{wire}} + 0.5 \cdot R_{\text{wire}} \cdot C_{\text{wire}} + R_{\text{wire}} \cdot C_L + R_p \cdot C_L$$

where $R_{\text{wire}} = r \cdot L$, $C_{\text{wire}} = c \cdot L$. The 0.38 and 0.5
coefficients come from the distributed RC nature of the wire (pi-model approximation).
The four terms capture: gate driving wire capacitance, wire driving its own capacitance,
wire driving load, and gate driving load.

### 11.3 Worked Example

A 7nm inverter ($R_p = 10\,\text{k}\Omega$) drives a 1mm metal-4 wire
($r = 0.2\,\Omega/\mu\text{m} = 200\,\Omega/\text{mm}$, $c = 0.2\,\text{fF}/\mu\text{m}$)
into a fanout of 4 ($C_L = 4 \times 0.5\,\text{fF} = 2\,\text{fF}$).

```text
Rwire = 0.2 Ω/μm × 1000 μm = 200 Ω
Cwire = 0.2 fF/μm × 1000 μm = 200 fF = 0.2 pF

t = 0.38 × 10k × 0.2pF + 0.5 × 200 × 0.2pF + 200 × 2fF + 10k × 2fF
  = 0.76ps + 20fs + 0.4fs + 20fs
  ≈ 0.80 ps
```

The gate resistance dominates; the wire contribution is negligible at 1mm but grows
quadratically with length ($RC \propto L^2$), making wire delay dominant beyond ~5mm.

---

## 11. Logical Effort

### 12.1 Methodology

Logical effort provides a quick delay estimation without full SPICE simulation. Define:

- $g$ (logical effort): how much worse a gate's input capacitance is vs an inverter for
  the same output current. $g_{\text{inv}} = 1$.
- $h = C_{\text{out}} / C_{\text{in}}$ (electrical effort or fanout)
- $p$ (parasitic delay): delay due to the gate's own parasitic capacitance
- $d = g \cdot h + p$ (normalized delay in units of $\tau$, the delay of a
  fanout-of-1 inverter)

### 12.2 Logical Effort Values for Common Gates

| Gate | $g$ (rising) | $g$ (falling) | $g$ (average) | $p$ |
|------|------|------|------|------|
| Inverter | 1 | 1 | 1 | 1 |
| 2-input NAND | 4/3 | 4/3 | 4/3 | 2 |
| 3-input NAND | 5/3 | 5/3 | 5/3 | 3 |
| 2-input NOR | 5/3 | 5/3 | 5/3 | 2 |
| 3-input NOR | 7/3 | 7/3 | 7/3 | 3 |
| 2-input XOR | 4 | 4 | 4 | 4 |
| MUX (2:1) | 2 | 2 | 2 | 2 |

Derivation for NAND2: the series PMOS stack needs 2× width to match the inverter's
pull-up current, giving input cap = 2 (P) + 2 (N) = 4 vs inverter's 2 (P) + 1 (N) = 3.
$g = 4/3$.

### 12.3 Path Delay Optimization

For a path of $N$ stages:

$$D = \sum_{i=1}^{N} (g_i \cdot h_i + p_i) = G \cdot H \cdot \prod h_i^{g_i} + P$$

where $G = \prod g_i$, $H = C_{\text{load}} / C_{\text{in}}$ (path electrical effort),
$P = \sum p_i$.

Optimal stage effort: $f^* = (G \cdot H)^{1/N}$, giving minimum delay
$D^* = N \cdot f^* + P$.

The key insight: minimum delay occurs when every stage bears equal effort $f^*$,
regardless of gate type. This determines optimal gate sizes via
$C_{\text{in},i} = g_i \cdot C_{\text{out},i} / f^*$.

### 12.4 Worked Example

Design a path from a flip-flop output through a NAND2 and an inverter to drive a load
of 20 unit inverters. Input capacitance budget: 1 unit inverter.

```text
G = g_NAND2 × g_inv = (4/3) × 1 = 4/3
H = 20 / 1 = 20  (no branching)
N = 2,  P = 2 + 1 = 3

Optimal per-stage effort:
  f* = √(G × H) = √(4/3 × 20) = √26.67 ≈ 5.16

Minimum path delay:
  D* = 2 × 5.16 + 3 = 13.32τ

Gate sizing:
  C_in,NAND2 = 1  (given, fixed by FF output drive strength)
  C_in,inv = g_NAND2 × C_in,NAND2 / f* = (4/3 × 1) / 5.16 ≈ 0.259 unit caps

Verification:
  h_NAND2 = C_in,inv / C_in,NAND2 = 0.259 / 1 = 0.259
  h_inv   = C_load / C_in,inv = 20 / 0.259 ≈ 77.2

Per-stage delays:
  d_NAND2 = g_NAND2 × h_NAND2 + p_NAND2 = (4/3)(0.259) + 2 = 2.35τ
  d_inv   = g_inv × h_inv + p_inv = (1)(77.2) + 1 = 78.2τ
  D_total = 2.35 + 78.2 = 80.5τ

This is far above D* = 13.32τ because H = 20 is too large for only 2 stages.
The problem needs more stages. Adding 2 more inverters (N = 4):

  G = (4/3) × 1 × 1 × 1 = 4/3
  H = 20
  f* = (G × H)^(1/4) = (4/3 × 20)^(0.25) = (26.67)^0.25 ≈ 2.27
  P = 2 + 1 + 1 + 1 = 5
  D* = 4 × 2.27 + 5 = 14.08τ

Gate sizes (working backwards from load):
  C_in,inv3 = g_inv × C_load / f* = 1 × 20 / 2.27 = 8.81
  C_in,inv2 = g_inv × C_in,inv3 / f* = 1 × 8.81 / 2.27 = 3.88
  C_in,inv1 = g_inv × C_in,inv2 / f* = 1 × 3.88 / 2.27 = 1.71
  C_in,NAND2 = g_NAND2 × C_in,inv1 / f* = (4/3) × 1.71 / 2.27 = 1.01 ≈ 1 ✓

D_total = 4 × (g_i × h_i) + P = 4 × 2.27 + 5 = 14.08τ  (matches D*)
```

The delay of 13.32τ is the theoretical minimum for this path topology; any other sizing
(unequal stage efforts) yields higher total delay.

---

## 12. 6T SRAM Cell

### 13.1 Cell Schematic

The 6-transistor SRAM cell is the fundamental building block of on-chip SRAM arrays.

```ascii-graph
          VDD                VDD
           |                  |
        ┌──┤ M1 (PMOS)    ┌──┤ M3 (PMOS)
        │  ├─┐            │  ├─┐
        │  │ Q  ──────────│──│ Qb   Cross-coupled inverters
        │  └─┘            │  └─┘     (M1/M2 and M3/M4)
        └──┤ M2 (NMOS)    └──┤ M4 (NMOS)
           │                  │
        ┌──┤ M5 (access)   ┌──┤ M6 (access)
        │  │                │  │
        BL                  BLB
           WL (wordline, gates of M5 and M6)
```

- M1/M2: first inverter (Q drives Qb)
- M3/M4: second inverter (Qb drives Q)
- M5/M6: access transistors gated by wordline WL, connecting storage nodes Q/Qb to
  bitlines BL/BLB

### 13.2 Read Operation

1. Precharge: BL and BLB are charged to VDD (precharge transistors ON)
2. Assert WL: M5 and M6 turn on
3. One bitline discharges through the access transistor and the "0" side:
   - If Q = 0, Qb = 1: BL discharges through M5 (access) and M2 (pull-down) BLB stays high (M6 connects to Qb = 1 = VDD)
   - Differential voltage ΔV = V(BL) - V(BLB) develops
4. Sense amplifier detects ΔV and amplifies to full-rail output
   - Sense amp offset ≈ 10-30 mV (determines minimum detectable ΔV)
5. WL deasserted, bitlines precharged for next access

- **Read time** — t_read ≈ C_BL × ΔV / I_cell
where C_BL ≈ 1-5 pF (for 256-row array), I_cell = read current through M5+M2

### 13.3 Write Operation

1. Drive bitlines: set BL and BLB to opposite values
   - To write Q = 0: BL → 0, BLB → VDD
   - To write Q = 1: BL → VDD, BLB → 0
2. Assert WL: M5 and M6 turn on
3. Access transistors must overpower the cross-coupled inverters to flip the state
   - The access transistor connected to the "1" side pulls it toward 0
   - Once the "1" node drops below the inverter trip point, positive feedback completes the flip
4. WL deasserted, new state is latched

Write margin: depends on relative strength of access transistors vs pull-up PMOS
   - Write requires: I(M5 or M6) > I(M1 or M3)
   - Stronger access transistors improve write margin

### 13.4 Static Noise Margin (SNM)

SNM is the maximum noise voltage tolerated without flipping the cell.

```text
Graphically: the largest square that fits inside the "butterfly curves"
  (superimposed VTC of the two cross-coupled inverters, one plotted normally
   and one with axes swapped)

SNM values (typical):
  Hold SNM  ≈ 0.4 × VDD   (WL = 0, cell isolated from bitlines)
  Read SNM  ≈ 0.2 × VDD   (WL = 1, access transistors degrade stability)
  Write margin ≈ 0.3 × VDD

Read SNM is lower than hold because the access transistor creates a voltage
divider with the pull-down NMOS, raising the "0" node voltage during read.
```

### 13.5 Read Stability vs Write Margin Tradeoff

```ascii-graph
This is the fundamental sizing tension in 6T SRAM:

  Access transistor strength (M5/M6):
    ↑ stronger → better write margin (can overpower inverters more easily)
    ↑ stronger → worse read stability (more disturbance of stored "0" during read)

  Pull-down NMOS strength (M2/M4):
    ↑ stronger → better read stability (holds "0" against access transistor)
    ↑ stronger → worse write margin (harder to flip)

  Pull-up PMOS strength (M1/M3):
    ↑ stronger → worse write margin (fights access transistor pulling "1" to "0")
    ↑ stronger → better hold SNM

Typical sizing ratio (β-ratio):  β = (W/L)_pull-down / (W/L)_access ≈ 1.5-3.0
  - Higher β → better read stability but harder writes
  - This ratio is the primary design knob for SRAM cell stability
```

### 13.6 Key Numbers

```text
6T cell area:         ≈ 0.04-0.08 μm² at N5
Bitline capacitance:  ≈ 1-5 pF for 256-row array
Sense amplifier offset: ≈ 10-30 mV
Minimum supply (data retention): ≈ 0.3-0.4V (VDD_min for hold)
Array efficiency:     ≈ 60-70% (cell area / total SRAM area)
Typical array size:    128-512 rows × 64-256 columns per subarray
```

---

## 13. Leakage Current Breakdown

### 14.1 Four Leakage Components

```text
Total leakage: I_total = I_sub + I_gate + I_junc + I_GIDL

At advanced nodes (N5): total ≈ 10-100 nA/μm per device
```

### 14.2 Subthreshold Leakage

```text
I_sub = I0 × exp[(VGS - Vth) / (n × Vt)] × [1 - exp(-VDS / Vt)]

where:
  I0 = μ0 × Cox × (W/L) × Vt² × exp(-Vth / (n × Vt))  (off-current prefactor)
  n  = 1 + Cd/Cox  (subthreshold slope factor, ≈ 1.3-1.6)
  Vt = kT/q ≈ 26 mV at 300K  (thermal voltage)

Dominant at advanced nodes: ~60-80% of total leakage
  - Increases exponentially as Vth decreases
  - Increases with temperature (Vth decreases with T)
  - DIBL makes it worse: effective Vth drops at high VDS

Mitigation: high-Vth transistors (HVT), power gating (sleep transistors),
  body biasing (reverse bias increases Vth)
```

### 14.3 Gate Oxide Leakage

```ascii-graph
I_gate = A × (Vox / t_ox)² × exp(-B × t_ox / Vox)

where:
  A, B: process-dependent constants (material-dependent)
  Vox: voltage across oxide
  t_ox: oxide thickness (or EOT for high-k)

Mechanism: quantum mechanical tunneling through thin gate dielectric
  - Direct tunneling dominant when t_ox < 3 nm
  - Fowler-Nordheim tunneling at thicker oxides / higher fields

High-k dielectrics (HfO2, k ≈ 20-25) reduce I_gate by allowing thicker
physical thickness for the same EOT:
  EOT = t_physical × (k_SiO2 / k_highk) = t_physical × (3.9 / 25)
  Example: 1.5 nm physical HfO2 → EOT = 1.5 × 3.9/25 ≈ 0.23 nm

Now ~5-10% of total leakage (was > 50% before high-k adoption at 45nm)
```

### 14.4 Junction (Reverse-Bias Diode) Leakage

```text
I_junc = Js × A_junction × [exp(qV / kT) - 1]

where:
  Js: reverse saturation current density (material and doping dependent)
  A_junction: junction area (source/drain diffusion area)

Always present when source or drain junction is reverse-biased relative to body.
  - In normal operation, at least one junction is always reverse-biased
  - Band-to-band tunneling (BTBT) adds to junction leakage at high doping

~5-10% of total leakage
```

### 14.5 GIDL (Gate-Induced Drain Leakage)

```text
I_GIDL ∝ exp(-B × Eg / (VDD - Vth))

where:
  Eg: silicon bandgap (≈ 1.12 eV at 300K)
  B: process-dependent constant

Mechanism: occurs in the gate-drain overlap region where high vertical field
  causes band-to-band tunneling. The gate creates a deep depletion region at
  the drain edge, and if the field is strong enough, valence band electrons
  tunnel to the conduction band.

Increases at higher VDD (higher field in overlap region).
~5-10% of total leakage

Mitigation: careful overlap engineering, lower VDD, LDD (lightly-doped drain) structures
```

---

## 14. Numbers to Memorize

| Quantity | Value | Why it matters |
|----------|-------|----------------|
| VDD at N5/N3 | 0.65-0.7V | Supply voltage for advanced nodes |
| Vth (typical) | 0.3-0.5V | Threshold voltage range |
| Electron mobility (bulk Si) | ~400 cm²/Vs | NMOS drive current reference |
| Hole mobility (bulk Si) | ~180 cm²/Vs | PMOS ~2.2x weaker |
| Subthreshold swing (60°C) | ~63 mV/decade | Minimum theoretical at room temp: 60 mV/dec |
| FO4 delay at N5 | ~12-15 ps | Gate delay normalization unit |
| EOT at N5 | ~0.8-1.0 nm | Equivalent oxide thickness |
| Fin pitch (N5) | ~25 nm | FinFET minimum feature |
| Fin height (N5) | ~50-60 nm | Determines drive current per fin |
| CMOS inverter gain (mid-region) | -gm × (ron‖rop) | Peak voltage gain |
| Noise margin (typical) | ~0.4 × VDD | Approximate for symmetrical inverter |
| 6T SRAM cell area (N5) | 0.04-0.08 μm² | Memory density driver |
| Subthreshold leakage (N5, per μm) | 10-100 nA/μm | Leakage power budget |
